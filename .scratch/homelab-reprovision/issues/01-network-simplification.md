# 01 — Network simplification (pre-reprovision prep)

**What to build:** Simplify the network to a flat 10.0.0.0/24 setup before tearing down the cluster. This prevents lockout if the Omada controller (currently running on the cluster) goes down during reprovision.

**Blocked by:** None — can start immediately (manual prep work)

**Status:** done (manual, pre-session)

## Why

- Omada controller currently runs on K8s — tearing down the cluster kills network management
- VLANs add complexity during initial debugging
- Flat network is simpler to troubleshoot during bootstrap
- Can re-add VLANs and Omada controller after cluster is stable

## Acceptance Criteria

- [x] Document current network config (VLANs, DHCP reservations, firewall rules)
- [x] Set Omada router (TL-ER605) to standalone mode or factory config with basic routing
- [x] Set Omada switch (TL-SG2008P) to standalone mode (unmanaged)
- [x] Set Omada AP (TL-EAP225) to standalone mode
- [x] Configure flat network: 10.0.0.0/24, gateway 10.0.0.1
- [x] Set static IPs or DHCP reservations for nodes (peggy, yelena, xialing)
- [x] Verify: All devices on LAN can reach each other
- [x] Verify: Internet access works
- [x] Verify: Can proceed with provisioning

### Notes

Network simplified before reprovisioning session. Flat 10.0.0.0/24 network with DHCP reservations for cluster nodes.
