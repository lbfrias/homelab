# 08 — PiHole DNS

**What to build:** PiHole running as the network's DNS server with a stable MetalLB IP.

**Blocked by:** 05 — MetalLB, 06 — Longhorn

**Status:** ready-for-agent

- [ ] Create PiHole deployment in `manifests/apps/network/`
- [ ] Configure LoadBalancer service (target IPs: 10.0.0.98, 10.0.0.99 per CONTEXT.md)
- [ ] Create PVC for PiHole config
- [ ] Migrate custom DNS entries from old repo if applicable
- [ ] Verify: DNS queries to PiHole IP resolve correctly
- [ ] Verify: Ad-blocking works
