# Physical Media Provisioning (Cloud-init + Kickstart)

Physical media with automated first-boot config for OS installation. RPi uses SD card with cloud-init, x86 uses USB installer with Kickstart.

**Context:** How to install OS on nodes for full automation.

**Decision:**
- **RPi (peggy, yelena):** SD card with Raspberry Pi OS + cloud-init
- **x86 (xialing):** USB installer with Rocky Linux + Kickstart (internal SSD)

**Rationale:**
- Simpler than PXE (no DHCP proxy, no boot server)
- RPi: cloud-init embedded in boot partition works well
- x86: internal SSD can't be flashed from desktop; USB installer + Kickstart automates install
- Both achieve same outcome: boot → Ansible-ready
