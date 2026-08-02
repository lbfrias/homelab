# Mixed OS (RPi OS + Rocky Linux)

Raspberry Pi OS Lite (64-bit) on ARM nodes (peggy, yelena), Rocky Linux 10 on x86 node (xialing). This provides best hardware support for each platform while giving exposure to RHEL-family administration.

**Rationale:**
- RPi OS has best hardware support for Raspberry Pi
- Rocky Linux provides RHEL practice (used at work)
- ARM64 support for Rocky on RPi is limited
- Ansible roles practice with multi-OS management
