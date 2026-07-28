# Step 5.1 — MetalLB

**What to build:** MetalLB providing LoadBalancer IPs on the LAN (10.0.0.30-99 pool).

**Blocked by:** Step 4 (Flux)

**Status:** ready

## Tasks

- [ ] Add MetalLB HelmRelease to `manifests/infrastructure/controllers/`
- [ ] Create IPAddressPool (10.0.0.30-10.0.0.99)
- [ ] Create L2Advertisement
- [ ] Verify: Test LoadBalancer service gets an IP from the pool
- [ ] Verify: Service reachable from LAN
