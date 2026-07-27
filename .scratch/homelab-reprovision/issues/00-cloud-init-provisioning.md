# 00 — OS provisioning via cloud-init and Kickstart

**What to build:** Flash OS images with automated first-boot configs so all nodes are ready for Ansible on first boot.

**Blocked by:** None — can start immediately

**Status:** done

## Workflow

### Raspberry Pi (cloud-init)
1. Download Raspberry Pi OS Lite image
2. Flash to SD card via `rpi-imager`
3. Add cloud-init files to boot partition
4. Boot node — cloud-init configures user, SSH keys, hostname, HDD /var mount
5. Ansible can connect immediately

### Rocky Linux (Kickstart)
1. Download Rocky Linux 10 installer ISO
2. Write ISO to USB drive
3. Add Kickstart config (ks.cfg) to USB
4. Boot xialing from USB with Kickstart parameter
5. Installer runs unattended, writes to internal SSD
6. Reboot — Ansible can connect immediately

## Acceptance Criteria

### Raspberry Pi (peggy, yelena) — ✅ DONE
- [x] Download Raspberry Pi OS Lite 64-bit image
- [x] Flash to SD cards via playbook
- [x] Add `user-data` and `meta-data` to boot partition for each node
- [x] Configure: hostname, user, SSH authorized key, passwordless sudo, HDD /var mount
- [x] Verify: Boot peggy, SSH in with key
- [x] Verify: Boot yelena, SSH in with key
- [x] Verify: HDD mounted as /var on both nodes

### Rocky Linux (xialing) — ✅ DONE
- [x] Download Rocky Linux 10 installer ISO
- [x] Create Kickstart template (ks.cfg.j2)
- [x] Configure: partitioning, hostname, user, SSH authorized key, passwordless sudo
- [x] Write ISO + ks.cfg to USB drive (playbook with mkksiso + EFI patch)
- [x] Boot xialing from USB — Kickstart runs unattended
- [x] Verify: Install completes unattended
- [x] Verify: SSH in with key after reboot (`ssh -p <port> <user>@<xialing-ip>`)

### Notes

**mkksiso EFI bug:** The `-R` and `--ks` flags update grub.cfg in the ISO filesystem but NOT the appended EFI partition. Fixed by post-flash step that copies correct grub.cfg to USB EFI partition. See: https://forums.rockylinux.org/t/problems-using-mkksiso-cmdline/19515/2

**Commit:** `f9182c5` — feat(provisioning): add Rocky Linux Kickstart support with EFI boot fix
