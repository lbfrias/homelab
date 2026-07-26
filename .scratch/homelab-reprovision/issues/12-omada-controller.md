# 11 — Omada Controller with macvlan

**What to build:** Omada Controller running with a macvlan interface on the LAN for L2 AP adoption.

**Blocked by:** 06 — Longhorn, 07 — External Secrets, 08 — Multus/macvlan

**Status:** ready-for-agent

- [ ] Create Omada Controller deployment in `manifests/apps/network/`
- [ ] Add macvlan annotation for LAN interface
- [ ] Configure static IP via Whereabouts annotation (finalize IP in CONTEXT.md)
- [ ] Create PVC for controller data
- [ ] Configure inform URL to macvlan IP
- [ ] Verify: Omada web UI accessible
- [ ] Verify: APs are adopted via L2 discovery
