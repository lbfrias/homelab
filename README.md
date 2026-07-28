# Homelab

Infrastructure as Code for my personal homelab — a 3-node K3s cluster with GitOps, automated provisioning, and full disaster recovery capability.

## Five-Step Provisioning

| Step | Name | Command | Description |
|------|------|---------|-------------|
| 1 | **Provision** | Flash media, boot | OS install via cloud-init (RPi) or Kickstart (x86) |
| 2 | **Bootstrap** | `ansible-playbook playbooks/bootstrap.yaml` | Packages, kernel modules, NFS/SMB |
| 3 | **K3s** | `ansible-playbook playbooks/k3s.yaml` | HA K3s cluster with embedded etcd |
| 4 | **Flux** | `ansible-playbook playbooks/flux.yaml` | GitOps bootstrap watching `manifests/` |
| 5 | **Services** | Automatic | Flux reconciles infrastructure and apps |

### Quick Start

```bash
# After nodes are provisioned (Step 1), from ansible/ directory:

# Run all steps (2-4) at once:
ansible-playbook -e @vars.local.yaml site.yaml

# Or run each step individually:
ansible-playbook -e @vars.local.yaml playbooks/bootstrap.yaml  # Step 2
ansible-playbook -e @vars.local.yaml playbooks/k3s.yaml        # Step 3
ansible-playbook -e @vars.local.yaml playbooks/flux.yaml       # Step 4

# Step 5 is automatic — Flux reconciles manifests/
```

## Hardware

| Node | Hardware | OS | Role |
|------|----------|-----|------|
| peggy | RPi 4 4GB | Raspberry Pi OS Lite | K3s control plane + worker |
| yelena | RPi 4 8GB | Raspberry Pi OS Lite | K3s control plane + worker |
| xialing | HP Prodesk i5-8500, 16GB RAM | Rocky Linux 10 | K3s control plane + worker, NAS |

## Services

- **Media:** Jellyfin (hardware transcoding), Radarr, Sonarr, Prowlarr, Transmission
- **Home Automation:** Home Assistant (with mDNS device discovery)
- **Network:** PiHole (DNS), Omada Controller, Tailscale
- **Monitoring:** Prometheus + Grafana (planned)

## Repository Structure

```
homelab/
├── ansible/
│   ├── playbooks/
│   │   ├── bootstrap.yaml   # Step 2
│   │   ├── k3s.yaml         # Step 3
│   │   └── flux.yaml        # Step 4
│   ├── provisioning/        # Step 1 templates
│   └── roles/
├── manifests/               # Step 5 (Flux-managed)
│   ├── infrastructure/
│   └── apps/
└── docs/
```

## Documentation

- [CONTEXT.md](./CONTEXT.md) — Architecture decisions and shared understanding
- [docs/restore-guide.md](./docs/restore-guide.md) — Disaster recovery procedures
- [docs/secrets.md](./docs/secrets.md) — Required secrets for reproduction

## Secrets

This repo uses External Secrets Operator with Bitwarden. If you're reproducing this setup, see [docs/secrets.md](./docs/secrets.md) for the list of required secrets you'll need to create.

## License

MIT
