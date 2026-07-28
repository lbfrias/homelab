# Step 5.4 — Multus/Macvlan

**What to build:** Pods can get secondary interface on LAN via macvlan (for Home Assistant, Omada).

**Blocked by:** Step 4 (Flux)

**Status:** ready

## Tasks

- [ ] Install Multus CNI (thick plugin, K3s paths)
- [ ] Install Whereabouts IPAM
- [ ] Create NetworkAttachmentDefinition for LAN (10.0.0.0/24)
- [ ] Create NetworkAttachmentDefinition for IoT VLAN (10.0.30.0/24)
- [ ] Deploy macvlan-shim DaemonSet
- [ ] Verify: Test pod with macvlan gets LAN IP
- [ ] Verify: Node can ping macvlan pod
- [ ] Document parent interface per node in CONTEXT.md
