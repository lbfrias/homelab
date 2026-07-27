# 00 — OS provisioning via cloud-init

**What to build:** Flash OS images with cloud-init configs so all nodes are ready for Ansible on first boot. No PXE server needed.

**Blocked by:** None — can start immediately

**Status:** ready-for-agent

## Workflow

1. Download OS images (Raspberry Pi OS Lite, Rocky Linux cloud image)
2. Flash to media (SD cards for Pis, USB/SSD for xialing)
3. Add cloud-init files to boot partition
4. Boot nodes — cloud-init configures user, SSH keys, hostname
5. Ansible can connect immediately

## Acceptance Criteria

### Raspberry Pi (peggy, yelena)
- [ ] Download Raspberry Pi OS Lite 64-bit image
- [ ] Flash to SD cards
- [ ] Add `user-data` and `meta-data` to boot partition for each node
- [ ] Configure: hostname, user, SSH authorized key, passwordless sudo
- [ ] Verify: Boot peggy, SSH in with key after ~2 min
- [ ] Verify: Boot yelena, SSH in with key after ~2 min

### Rocky Linux (xialing)
- [ ] Download Rocky Linux 10 cloud image (GenericCloud)
- [ ] Flash to USB or SSD
- [ ] Add cloud-init config (NoCloud datasource)
- [ ] Configure: hostname, user, SSH authorized key, passwordless sudo
- [ ] Verify: Boot xialing, SSH in with key

## cloud-init template

```yaml
#cloud-config
hostname: <nodename>
users:
  - name: <username>
    ssh_authorized_keys:
      - <your-ssh-public-key>
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: true
```
