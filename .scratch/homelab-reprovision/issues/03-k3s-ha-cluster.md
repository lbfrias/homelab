# 02 — K3s HA cluster bootstrap

**What to build:** A 3-node K3s cluster with all nodes as control plane (HA embedded etcd). Start with vanilla K3s install, debug and fix issues as they appear.

**Blocked by:** 02 — Ansible inventory + SSH + NFS/SMB

**Status:** done

- [x] Create `ansible/playbooks/k3s.yml` that installs K3s
- [x] First node (xialing) uses `--cluster-init`
- [x] Other nodes join via `--server https://xialing:6443`
- [x] Disable servicelb and traefik (will use MetalLB, no ingress needed)
- [x] Handle K3s token (prompt or pull from first node)
- [x] Verify: `kubectl get nodes` shows all 3 nodes Ready
- [x] Document any fixes needed (iptables-legacy, kernel-modules-extra, firewalld, etc.) in CONTEXT.md
