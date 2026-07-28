# Step 1 — Provision

**What to build:** Flash OS images so nodes boot ready for Ansible.

**Blocked by:** Step 0 (Network Prep)

**Status:** done

## Workflow

### Raspberry Pi (cloud-init)
1. Flash Raspberry Pi OS Lite to SD card
2. Add cloud-init files (`user-data`, `meta-data`) to boot partition
3. Boot — cloud-init `runcmd` handles everything:
   - `wipefs` → wipe HDD
   - `parted` → create GPT partition
   - `mkfs.ext4` → format partition
   - `rsync` → copy /var to HDD
   - Update fstab with UUID, reboot
4. Second boot — fstab mounts HDD to /var, node ready for Ansible

**Important:** No `package_update`/`package_upgrade` in cloud-init. Ansible handles packages after /var migration is complete.

### Rocky Linux (Kickstart)
1. Write Rocky Linux 10 ISO + ks.cfg to USB
2. Boot xialing from USB with Kickstart parameter
3. Installer writes to internal SSD unattended, reboots ready for Ansible

## Tasks

### Raspberry Pi (peggy, yelena)
- [x] Flash Raspberry Pi OS Lite 64-bit to SD cards
- [x] Add `user-data` and `meta-data` to boot partition
- [x] Configure: hostname, user, SSH key, passwordless sudo
- [x] Fix cloud-init template: all HDD setup in runcmd (wipefs, parted, mkfs, rsync)
- [x] Fix cloud-init template: remove package_update/package_upgrade
- [x] Reprovision peggy and yelena with fixed cloud-init
- [x] Verify: SSH in with key, /var mounted on HDD

### Rocky Linux (xialing)
- [x] Create Kickstart template (ks.cfg.j2)
- [x] Configure: partitioning, hostname, user, SSH key, passwordless sudo
- [x] Write ISO + ks.cfg to USB drive
- [x] Apply EFI boot fix (mkksiso bug workaround)
- [x] Verify: Install completes unattended, SSH works

## Notes

**mkksiso EFI bug:** The `-R` and `--ks` flags update grub.cfg but NOT the appended EFI partition. Fixed by post-flash step that copies correct grub.cfg to USB EFI partition.

**RPi /var migration:** cloud-init's `disk_setup`/`fs_setup` modules skip formatting if disk layout "matches" (even with `overwrite: true`). Solution: do all HDD setup in `runcmd` with explicit commands (wipefs, parted, mkfs, rsync).
