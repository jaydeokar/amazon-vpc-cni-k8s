# Assign the host veth MAC without listing the node's links

Stop pod ADDs from failing under concurrent pod creation (#3576) by assigning the host-side veth a random MAC without first listing every link on the node to check that the address is unused.

## Motivation

Since #3354 (v1.20.1), `setupVeth` picks the host veth's MAC by drawing a random address and comparing it against every link on the node (`generateUniqueRandomMAC`, `LinkList`). The explicit MAC itself is needed for a different reason: it marks the link `NET_ADDR_SET` so udev's `MACAddressPolicy=persistent` doesn't rewrite it after the plugin has pinned the pod's gateway neighbor entry to it. The uniqueness check was never required for that.

Under concurrent ADDs, each pod's veth creation interrupts the others' `LinkList` dumps. #3440 (v1.20.5) retries up to 5 times; once exhausted, the ADD fails:

```
failed to generate Unique MAC addr for host side veth: netlink operation interruption persisted after 5 attempts
```

The kubelet retries ~10s later. Reporters on #3576/#3932 measured 2–22 failures per 192-pod burst on a 192-vCPU node, 0 with the dump removed; a standalone repro (N goroutines doing `LinkList` + veth create) showed 31% failing at N=192, 0% without the dump. The check also never guaranteed uniqueness — two concurrent ADDs can still draw the same address after both dumps return.

### Non-Goals

- **Making host veth MACs unique.** Nothing needs them to be unique — see "Why this is safe" below.
- **Other netlink dumps in ipamd** (reconcile, ENI setup). They don't run per pod.

## Proposal

No new flag. `setupVeth` assigns a random locally administered MAC (`generateRandomMAC`) without listing the node's links. `linkByMac` (ipamd) and `getNodePrimaryIF` (`aws-vpc-cni-init`) — the two places that look up an interface *by* MAC — now skip veth links, so neither can mistake a pod's veth for an ENI.

### Why this is safe

Host veths are never bridged or promiscuous (no `LinkSetMaster`/`SetPromisc`/macvlan anywhere in this codebase), so no shared L2 domain ever has to pick between two interfaces by MAC. Delivery across a veth is by kernel pointer, not MAC lookup; the pod's gateway is a pinned permanent neighbor entry (read, not searched); the host routes to a pod by ifindex, not MAC; and L2 headers are rebuilt at every hop, so a veth's MAC never reaches the ENI or the VPC. Two pods sharing a host-veth MAC is indistinguishable from today's pre-#3354 behavior, where the kernel assigned it with no check at all

The one real risk is a veth's MAC colliding with an *ENI's* MAC, which could feed `linkByMac` the wrong link. The existing check only prevented this for an ENI already attached at veth-creation time (not one attached later, and racy regardless), so removing it doesn't meaningfully raise that already-negligible probability (~5×10⁻¹¹ for 250 pods / 15 ENIs). The `linkByMac`/`getNodePrimaryIF` veth filter removes the consequence entirely — a veth can never be returned by either lookup — which is a net improvement over today, where no such filter exists.

The Network Policy Agent attaches by `hostVethName`/ifindex (`TCEgressAttach`/`TCIngressAttach`), not MAC, so it's unaffected.

### Testing

- `Test_generateRandomMAC`: 1,000 draws are unicast/locally-administered.
- `TestLinkByMacSkipsHostVeth`, `TestLinkNameByMAC`: a veth carrying an ENI's MAC is listed first; the lookup still returns the ENI.
- Existing `setupVeth`/`SetupPodNetwork`/`SetupBranchENIPodNetwork` tests pass unchanged.
- Before release: repeat the #3576 burst (192 pods, IPv4 + IPv6) targeting 0 failures, with normal pod connectivity checks.

### Interaction with Existing Features

Security groups for pods, IPv6, and multi-NIC all go through the same `setupVeth` path and are unaffected — none of them key anything off the host veth's MAC.

## Alternatives Considered

1. **Tolerate `ErrDumpInterrupted`** (use the partial dump, like `netlinksafe`). Removes the failure but keeps a full link dump on every ADD, comparing against a list known to be incomplete.
2. **More retries / backoff.** Slower under exactly the load that causes the problem; doesn't remove it.
3. **Flag-gated skip (#3932).** Works, but leaves the failure on by default for a check that protects nothing. This proposal makes that behavior unconditional.
4. **Derive the MAC from the pod's IP or identity (container ID, namespace/name).** Any derivation gets truncated to the MAC's 46 usable bits, which collides exactly as often as a random draw — hashing buys nothing here. (The codebase gets real uniqueness for pod identity elsewhere, in `hostVethAlias`, precisely by keeping the full SHA-256 instead of truncating it — a MAC has no spare bits to do the same.)
5. **One fixed MAC for every veth**. Would also work; random keeps veths distinguishable in captures with no other change.
6. **Return ENI MACs in `AddNetworkReply` and re-draw on a match.** Only helps for ENIs already attached, duplicating what the `linkByMac` filter already covers.

## Security Considerations

No change to privileges or interfaces. MACs are drawn from `crypto/rand` as today. The `linkByMac`/`getNodePrimaryIF` filter ensures a pod's veth can never be configured as an ENI.

## Backward Compatibility

No config change. Existing veths are unaffected; only new ones skip the check. Plugin, ipamd, and init ship together, so both MAC-lookup filters land with the change. Rollback runs the old check again; an older ipamd/init without the veth filter is exposed only in the already-negligible collision case above.

## Open Questions

- **AL2023 `MACAddress=` matching.** If `amazon-ec2-net-utils`/systemd-networkd match ENIs by MAC alone (no driver/name condition), a colliding veth could in theory be picked up outside our code. Needs checking; Alternative 6 above is the fix if so.
