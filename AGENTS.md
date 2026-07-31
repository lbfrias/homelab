# Agent Instructions

Guidelines for AI agents working on this repository.

## Repository Structure

- `ansible/` — Node provisioning (Steps 1-4)
- `manifests/` — K8s workloads managed by Flux (Step 5)
- `.scratch/homelab-reprovision/issues/` — Local issue tracker
- `docs/` — User-facing documentation

## Five-Step Provisioning Model

| Step | Tool | Directory |
|------|------|-----------|
| 1. Provision | cloud-init/Kickstart | `ansible/provisioning/` |
| 2. Bootstrap | Ansible | `ansible/playbooks/bootstrap.yaml` |
| 3. K3s | Ansible | `ansible/playbooks/k3s.yaml` |
| 4. Flux | Ansible | `ansible/playbooks/flux.yaml` |
| 5. Services | Flux | `manifests/` |

## Configuration (DRY)

Shared values live in two files that **must stay in sync**:

| File | Used by |
|------|---------|
| `ansible/vars.yaml` | Ansible (Steps 1-4) |
| `manifests/infrastructure/controllers/cluster-vars.yaml` | Flux substitution (Step 5) |

K8s manifests use `${VAR}` syntax (e.g., `${XIALING_IP}`, `${LAN_CIDR}`) which Flux substitutes at reconciliation time from the `cluster-vars` ConfigMap.

**When changing IPs, subnets, or shared config:** Update both files.

**Note:** `apps/` is managed by a Flux Kustomization CR (in `infrastructure/kustomizations.yaml`), NOT directly in `manifests/kustomization.yaml`. This ordering ensures `cluster-vars` exists before substitution runs.

## Documentation Sync

**After every feature or change, sync documentation to match the repo state:**

1. **CONTEXT.md** — Update if the change affects:
   - Architecture decisions (add/update ADR)
   - Glossary terms
   - Conventions
   - Hardware/networking info

2. **README.md** — Update if the change affects:
   - Quick start commands
   - Services list
   - Repository structure

3. **Issue tickets** (`.scratch/`)  — Mark completed tasks, update status

4. **docs/** — Update relevant guides (restore-guide.md, secrets.md)

## Conventions

- YAML files use `.yaml` extension (not `.yml`)
- Container images should always be version locked
- Secrets are never committed — use Bitwarden Secrets Manager Operator
- Node names: peggy (RPi 4GB), yelena (RPi 8GB), xialing (x86 NAS)
