# Step 5.4 — Multus/Macvlan

**What to build:** Pods can get secondary interface on LAN via macvlan (for Home Assistant, Omada).

**Blocked by:** Step 4 (Flux)

**Status:** done

## Tasks

- [x] Install Multus CNI (via RKE2 Helm chart v4.3.008)
- [x] Install Whereabouts IPAM (subchart of rke2-multus)
- [x] Create NetworkAttachmentDefinition for LAN (10.0.0.0/24)
- [x] Create NetworkAttachmentDefinition for IoT VLAN (10.0.30.0/24)
- [x] ~~Deploy macvlan-shim DaemonSet~~ Not needed — RKE2 chart handles host-to-pod
- [x] Verify: Test pod with macvlan gets LAN IP (10.0.0.200)
- [x] Verify: Node can ping macvlan pod
- [x] Document parent interface per node in CONTEXT.md

## Implementation Notes

Used RKE2 Helm charts per K3s docs (docs.k3s.io/networking/multus-ipams):
- `rke2-multus` chart includes Whereabouts as subchart
- Key paths for K3s: `/var/lib/rancher/k3s/data/cni/` (binaries), `/var/lib/rancher/k3s/agent/etc/cni/net.d` (config)

Parent interfaces differ by node:
- RPi (peggy, yelena): `eth0` / `eth0.30`
- x86 (xialing): `eno1` / `eno1.30`

Separate NADs created per interface family: `macvlan-lan` / `macvlan-lan-x86`

## Pending: VLAN Interface Creation

The IoT VLAN NADs (`macvlan-iot`, `macvlan-iot-x86`) require VLAN interfaces on the host that **do not yet exist**:
- `eno1.30` on xialing (required for Home Assistant)
- `eth0.30` on peggy/yelena (optional, only if IoT workloads run there)

This must be added to Ansible bootstrap (Step 2) before deploying Home Assistant.
