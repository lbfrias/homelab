# 01 — Network simplification (pre-reprovision prep)

**What to build:** Simplify the network to a flat 10.0.0.0/24 setup before tearing down the cluster. This prevents lockout if the Omada controller (currently running on the cluster) goes down during reprovision.

**Blocked by:** None — can start immediately (manual prep work)

**Status:** ready-for-agent

## Why

- Omada controller currently runs on K8s — tearing down the cluster kills network management
- VLANs add complexity during initial debugging
- Flat network is simpler to troubleshoot during bootstrap
- Can re-add VLANs and Omada controller after cluster is stable

## Acceptance Criteria

- [ ] Document current network config (VLANs, DHCP reservations, firewall rules)
- [ ] Set Omada router (TL-ER605) to standalone mode or factory config with basic routing
- [ ] Set Omada switch (TL-SG2008P) to standalone mode (unmanaged)
- [ ] Set Omada AP (TL-EAP225) to standalone mode
- [ ] Configure flat network: 10.0.0.0/24, gateway 10.0.0.1
- [ ] Set static IPs or DHCP reservations for nodes (peggy, yelena, xialing)
- [ ] Verify: All devices on LAN can reach each other
- [ ] Verify: Internet access works
- [ ] Verify: Can proceed with PXE boot setup
