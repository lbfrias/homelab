# Step 5.5 — PiHole

**What to build:** PiHole as network DNS with stable MetalLB IP.

**Blocked by:** Step 5.1 (MetalLB), Step 5.2 (Longhorn)

**Status:** ready

## Tasks

- [ ] Create PiHole deployment in `manifests/apps/network/`
- [ ] Configure LoadBalancer (10.0.0.98, 10.0.0.99)
- [ ] Create PVC for PiHole config
- [ ] Migrate custom DNS entries if applicable
- [ ] Verify: DNS queries resolve correctly
- [ ] Verify: Ad-blocking works
