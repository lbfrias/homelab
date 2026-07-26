# 04 — MetalLB L2 mode

**What to build:** MetalLB providing LoadBalancer IPs on the LAN. Services can get stable IPs from the 10.0.0.30-99 pool.

**Blocked by:** 03 — Flux bootstrap

**Status:** ready-for-agent

- [ ] Add MetalLB HelmRelease to `manifests/infrastructure/controllers/`
- [ ] Create IPAddressPool (10.0.0.30-10.0.0.99)
- [ ] Create L2Advertisement
- [ ] Verify: Create a test LoadBalancer service, confirm it gets an IP from the pool
- [ ] Verify: Can reach the service from another machine on the LAN
