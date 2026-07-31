# Step 3 — K3s

**What to build:** 3-node K3s cluster with all nodes as control plane (HA embedded etcd).

**Command:** `ansible-playbook -e @vars.local.yaml playbooks/k3s.yaml`

**Blocked by:** Step 2 (Bootstrap)

**Status:** done

## Tasks

- [x] Create `ansible/playbooks/k3s.yaml` (Step 3 playbook)
- [x] First node (xialing) uses `--cluster-init`
- [x] Other nodes join via `--server https://xialing:6443`
- [x] Disable servicelb and traefik (built-in LB not needed; services use macvlan or NodePort)
- [x] Handle K3s token (prompt or pull from first node)
- [x] Fetch kubeconfig to `~/.kube/config-homelab`
- [x] Verify: `kubectl get nodes` shows all 3 nodes Ready
