# Step 5.8 — Omada Controller

**What to build:** Omada Controller with macvlan on LAN for L2 AP adoption.

**Blocked by:** Step 5.2 (Longhorn), Step 5.4 (Multus/Macvlan)

**Status:** ready

## Tasks

- [ ] Create Omada deployment in `manifests/apps/network/`
- [ ] Add macvlan annotation for LAN interface
- [ ] Configure static IP via Whereabouts
- [ ] Create PVC for controller data
- [ ] Configure inform URL to macvlan IP
- [ ] Verify: Omada web UI accessible
- [ ] Verify: APs are adopted via L2
