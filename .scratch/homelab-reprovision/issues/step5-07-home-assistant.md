# Step 5.7 — Home Assistant

**What to build:** Home Assistant with macvlan on IoT VLAN for mDNS device discovery.

**Blocked by:** Step 5.2 (Longhorn), Step 5.4 (Multus/Macvlan)

**Status:** ready

## Tasks

- [ ] Create Home Assistant deployment in `manifests/apps/home/`
- [ ] Add macvlan annotation for IoT VLAN interface
- [ ] Configure static IP via Whereabouts
- [ ] Create PVC for `/config`
- [ ] Verify: HA web UI accessible
- [ ] Verify: mDNS device discovery works
