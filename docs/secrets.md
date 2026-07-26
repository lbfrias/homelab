# Secrets Documentation

This homelab uses External Secrets Operator to pull secrets from Bitwarden. If you're reproducing this setup with your own secret store, you'll need to create the following secrets.

## Required Secrets

### Infrastructure

| Secret Name | Keys | Used By |
|-------------|------|---------|
| `k3s-token` | `token` | K3s cluster join token |
| `longhorn-backup-credentials` | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | Longhorn S3 backups (if using S3) |

### Networking

| Secret Name | Keys | Used By |
|-------------|------|---------|
| `tailscale-auth` | `authkey` | Tailscale operator |
| `pihole-admin` | `password` | PiHole web UI |

### Media Stack

| Secret Name | Keys | Used By |
|-------------|------|---------|
| `radarr-api-key` | `apikey` | Radarr API access |
| `sonarr-api-key` | `apikey` | Sonarr API access |
| `prowlarr-api-key` | `apikey` | Prowlarr API access |
| `transmission-credentials` | `username`, `password` | Transmission RPC |

### Home Automation

| Secret Name | Keys | Used By |
|-------------|------|---------|
| `home-assistant-secrets` | (various) | HA secrets.yaml |

## Manual Creation (without External Secrets)

If you're not using External Secrets Operator, create secrets manually:

```bash
# Example: Create radarr API key secret
kubectl create secret generic radarr-api-key \
  --namespace=media \
  --from-literal=apikey=YOUR_API_KEY_HERE

# Example: Create tailscale auth key
kubectl create secret generic tailscale-auth \
  --namespace=tailscale \
  --from-literal=authkey=tskey-auth-xxxxx
```

## Bitwarden Structure

For External Secrets with Bitwarden, organize secrets as:

```
Bitwarden Vault/
└── homelab/
    ├── k3s-token
    ├── tailscale-auth
    ├── pihole-admin
    ├── radarr-api-key
    ├── sonarr-api-key
    ├── prowlarr-api-key
    ├── transmission-credentials
    └── home-assistant-secrets
```

Each item should have custom fields matching the key names above.
