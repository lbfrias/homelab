# Multus + Macvlan for LAN Access

Use Multus CNI with macvlan secondary interface for pods needing direct LAN access. Pods get dual interfaces: eth0 (cluster) + net1 (LAN).

**Context:** Home Assistant needs mDNS for device discovery. Omada needs L2 access for AP adoption.

**Rationale:**
- mDNS/multicast requires direct LAN access
- Dual-interface preserves cluster connectivity
- Static IPs for infrastructure pods (HA, Omada)
- Macvlan-shim DaemonSet solves host-to-pod communication
