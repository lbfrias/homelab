# 10 — Home Assistant with macvlan

**What to build:** Home Assistant running with a macvlan interface on the IoT VLAN for mDNS device discovery.

**Blocked by:** 06 — Longhorn, 07 — External Secrets, 08 — Multus/macvlan

**Status:** ready-for-agent

- [ ] Create Home Assistant deployment in `manifests/apps/home/`
- [ ] Add macvlan annotation for IoT VLAN interface
- [ ] Configure static IP via Whereabouts annotation (finalize IP in CONTEXT.md)
- [ ] Create PVC for `/config`
- [ ] Create ExternalSecret for HA secrets if needed
- [ ] Verify: HA web UI accessible
- [ ] Verify: mDNS device discovery works (can find IoT devices)
