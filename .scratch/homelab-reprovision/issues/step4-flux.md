# Step 4 — Flux

**What to build:** Flux installed and reconciling from this repo. The GitOps foundation for all K8s workloads.

**Command:** `ansible-playbook -e @vars.local.yaml playbooks/flux.yaml`

**Blocked by:** Step 3 (K3s)

**Status:** ready

## Tasks

- [x] Create Flux directory structure under `manifests/`
- [x] Create `ansible/playbooks/flux.yaml` (Step 4 playbook)
- [x] Add `github_token`, `github_owner`, `github_repo` to `vars.local.yaml`
- [x] Run playbook to bootstrap Flux
- [x] Verify: `flux get all` shows healthy reconciliation
- [ ] Verify: Changes pushed to repo are automatically applied
