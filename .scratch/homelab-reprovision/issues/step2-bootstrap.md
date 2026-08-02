# Step 2 — Bootstrap

**What to build:** Update package cache, install common packages, configure NFS/SMB on xialing.

**Command:** `ansible-playbook -e @vars.yaml -e @secrets.local.yaml playbooks/bootstrap.yaml`

**Blocked by:** Step 1 (Provision)

**Status:** done

## What bootstrap.yaml does

1. Update package cache (apt/dnf)
2. **Upgrade all packages** (deferred from cloud-init to avoid dpkg corruption during /var migration)
3. Install OS-specific packages first (epel-release on Rocky, glances on Debian)
4. Install common packages: `curl`, `vim`, `jq`
5. Run `nfs_smb` role on xialing

**Note:** K3s prerequisites (kernel modules, sysctl, firewalld, cgroup) moved to `k3s.yaml` — they belong with K3s setup, not general bootstrap.

**Note:** Package update/upgrade must happen here, not in cloud-init. On RPis, cloud-init runs before /var migration completes — running apt before the migration corrupts dpkg state.

## Tasks

- [x] Create `ansible/inventory/hosts.yaml` with all 3 nodes
- [x] Group nodes: `rpi` (peggy, yelena) and `x86` (xialing)
- [x] Create `ansible/playbooks/ping.yaml` for connectivity check
- [x] Create `ansible/roles/common/` — minimal, just package cache + packages
- [x] Create `ansible/playbooks/bootstrap.yaml`
- [x] Packages: curl, vim, jq (glances on Debian only — not in EPEL for Rocky 10)
- [x] Run `nfs_smb` role on xialing
- [x] Verify: bootstrap.yaml succeeds on all nodes
- [x] Verify: NFS export accessible from cluster nodes
- [x] Verify: SMB share accessible

## Refactoring done

- Moved K3s prereqs (kernel modules, sysctl, firewalld, cgroup) from bootstrap.yaml to k3s.yaml
- Stripped common role to minimal: package cache + essential packages only
- Removed package installs from cloud-init — Ansible handles all packages now
- Updated common role to use `ansible_facts['os_family']` syntax (fixes deprecation warnings)

## Pending Tasks

- [x] Create VLAN interface `eno1.300` on xialing for IoT VLAN (required for Home Assistant macvlan)
