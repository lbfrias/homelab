# Step 2 — Bootstrap

**What to build:** Update package cache, install common packages, configure NFS/SMB on xialing.

**Command:** `ansible-playbook -e @vars.local.yaml playbooks/bootstrap.yaml`

**Blocked by:** Step 1 (Provision)

**Status:** in progress (blocked on Step 1 RPi fix)

## What bootstrap.yaml does

1. Update package cache (apt/dnf)
2. Install OS-specific packages first (epel-release on Rocky)
3. Install common packages: `curl`, `vim`, `jq`, `glances`
4. Run `nfs_smb` role on xialing

**Note:** K3s prerequisites (kernel modules, sysctl, firewalld, cgroup) moved to `k3s.yaml` — they belong with K3s setup, not general bootstrap.

## Tasks

- [x] Create `ansible/inventory/hosts.yaml` with all 3 nodes
- [x] Group nodes: `rpi` (peggy, yelena) and `x86` (xialing)
- [x] Create `ansible/playbooks/ping.yaml` for connectivity check
- [x] Create `ansible/roles/common/` — minimal, just package cache + packages
- [x] Create `ansible/playbooks/bootstrap.yaml`
- [x] Packages: curl, vim, jq, glances (epel-release first on Rocky)
- [x] Run `nfs_smb` role on xialing
- [ ] Verify: bootstrap.yaml succeeds on all nodes (blocked on Step 1)
- [ ] Verify: NFS export accessible from cluster nodes
- [ ] Verify: SMB share accessible

## Refactoring done

- Moved K3s prereqs (kernel modules, sysctl, firewalld, cgroup) from bootstrap.yaml to k3s.yaml
- Stripped common role to minimal: package cache + essential packages only
- Removed package installs from cloud-init — Ansible handles all packages now
