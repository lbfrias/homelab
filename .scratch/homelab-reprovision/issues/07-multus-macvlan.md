# 07 — Multus CNI + macvlan networking

**What to build:** Pods can get a secondary interface on the LAN via macvlan. Required for Home Assistant (mDNS) and Omada Controller (AP adoption).

**Blocked by:** 03 — Flux bootstrap

**Status:** ready-for-agent

- [ ] Install Multus CNI (thick plugin, K3s-specific paths)
- [ ] Install Whereabouts IPAM
- [ ] Create NetworkAttachmentDefinition for LAN (10.0.0.0/24)
- [ ] Create NetworkAttachmentDefinition for IoT VLAN (10.0.30.0/24) if needed
- [ ] Deploy macvlan-shim DaemonSet for node-to-pod communication
- [ ] Verify: Deploy test pod with macvlan annotation, confirm it gets LAN IP
- [ ] Verify: Node can ping the macvlan pod (via shim)
- [ ] Verify: Pod can reach LAN gateway
- [ ] Document parent interface name per node in CONTEXT.md
