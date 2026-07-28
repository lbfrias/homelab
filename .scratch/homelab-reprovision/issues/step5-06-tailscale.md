# Step 5.6 — Tailscale

**What to build:** Tailscale operator for remote cluster access.

**Blocked by:** Step 5.1 (MetalLB), Step 5.3 (External Secrets)

**Status:** ready

## Tasks

- [ ] Create Tailscale operator in `manifests/apps/network/`
- [ ] Create ExternalSecret for Tailscale auth key
- [ ] Configure subnet router if needed
- [ ] Verify: Cluster accessible remotely via Tailscale
