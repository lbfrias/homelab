# Step 5.7 — Home Assistant

**What to build:** Home Assistant with macvlan on IoT VLAN for mDNS device discovery.

**Blocked by:** Step 5.2 (Longhorn), Step 5.4 (Multus/Macvlan)

**Status:** done

## Tasks

- [x] Create Home Assistant deployment in `manifests/apps/home/`
- [x] Add macvlan annotation for IoT VLAN interface
- [x] Configure static IP via Whereabouts (10.0.30.220/24)
- [x] Create PVC for `/config` (5Gi on Longhorn)
- [x] Verify: HA web UI accessible (HTTP 302 via port-forward)
- [x] Verify: mDNS device discovery works (pinged IoT gateway 10.0.30.1)

## Implementation Notes

- Deployment pinned to xialing (only node with `eno1.300` for macvlan-iot-x86)
- Image: `ghcr.io/home-assistant/home-assistant:2026.7.4`
- Macvlan IP: `10.0.30.220/24` on IoT VLAN
- Service: ClusterIP on port 8123
- Need to restore backup via HA native backup feature
