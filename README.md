# Homelab

Infrastructure as Code for my personal homelab — a 3-node K3s cluster with GitOps, automated provisioning, and full disaster recovery capability.

## Overview

This repo contains everything needed to provision and manage my homelab from scratch:

- **Ansible** for node provisioning (OS config, K3s install, hardening)
- **Flux** for GitOps-based K8s workload management
- **Longhorn** for persistent storage with backups
- **Multus + Macvlan** for LAN-attached workloads (Home Assistant, Omada Controller)

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

## Quick Start

### Prerequisites

- Ansible installed on your workstation
- SSH access to all nodes
- Bitwarden CLI configured (for secrets)

### Provision Nodes

```bash
cd ansible
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

### Bootstrap Flux

```bash
flux bootstrap github \
  --owner=<your-github-username> \
  --repository=homelab \
  --path=manifests/clusters/homelab
```

## Documentation

- [CONTEXT.md](./CONTEXT.md) — Architecture decisions and shared understanding
- [docs/networking.md](./docs/networking.md) — Multus + Macvlan setup
- [docs/restore-guide.md](./docs/restore-guide.md) — Disaster recovery procedures
- [docs/secrets.md](./docs/secrets.md) — Required secrets for reproduction

## Secrets

This repo uses External Secrets Operator with Bitwarden. If you're reproducing this setup, see [docs/secrets.md](./docs/secrets.md) for the list of required secrets you'll need to create.

## License

MIT
