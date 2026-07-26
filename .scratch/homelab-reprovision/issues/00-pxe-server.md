# 00 — PXE server for OS provisioning

**What to build:** A PXE boot server (Docker on Linux desktop) that can netboot all 3 nodes and install their respective OSes automatically.

**Blocked by:** None — can start immediately

**Status:** ready-for-agent

## Setup

- Linux desktop on same LAN as nodes (10.0.0.0/24)
- Docker with `--network host` for DHCP proxy + TFTP
- netboot.xyz for boot menus (supports both ARM64 and x86)
- Proxy DHCP mode — Omada router keeps handling IP assignment

## Acceptance Criteria

- [ ] Run PXE server container (dnsmasq or netboot.xyz) with `--network host`
- [ ] Configure proxy DHCP mode (don't conflict with Omada DHCP)
- [ ] Serve netboot.xyz boot files for ARM64 and x86
- [ ] Create Kickstart file for Rocky Linux 10 (xialing)
- [ ] Raspberry Pi prep: Update EEPROM for network boot on peggy and yelena (one-time SD card boot)
- [ ] Verify: Boot peggy from network, OS installs
- [ ] Verify: Boot yelena from network, OS installs
- [ ] Verify: Boot xialing from network, Rocky 10 installs via kickstart
- [ ] Verify: Can SSH to all nodes after install (SSH key injected)
