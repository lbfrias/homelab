# 09 — Tailscale remote access

**What to build:** Tailscale operator running so the cluster is accessible remotely via Tailscale network.

**Blocked by:** 04 — MetalLB, 06 — External Secrets (for auth key)

**Status:** ready-for-agent

- [ ] Create Tailscale operator deployment in `manifests/apps/network/`
- [ ] Create ExternalSecret for Tailscale auth key
- [ ] Configure subnet router if needed
- [ ] Verify: Cluster accessible from remote machine via Tailscale
