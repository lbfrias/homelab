# Secrets Documentation

This homelab uses **Bitwarden Secrets Manager Operator** to sync secrets from Bitwarden SM. If you're reproducing this setup, you'll need to create the following secrets.

## Bitwarden Secrets Manager Setup

### Prerequisites

1. A Bitwarden organization with Secrets Manager enabled
2. A Machine Account with access to your secrets
3. An access token for the Machine Account

### Operator Installation

The operator is deployed via Flux from `manifests/infrastructure/controllers/bitwarden-sm-operator/`. It creates:
- Namespace: `sm-operator-system`
- CRD: `BitwardenSecret`

### Auth Token Secret

Create the auth token secret **once** in `flux-system` with Reflector annotations — it will be mirrored to all namespaces automatically:

```bash
kubectl create secret generic bw-auth-token \
  -n flux-system \
  --from-literal=token="<YOUR_ACCESS_TOKEN>"

kubectl annotate secret bw-auth-token -n flux-system \
  reflector.v1.k8s.emberstack.com/reflection-allowed="true" \
  reflector.v1.k8s.emberstack.com/reflection-auto-enabled="true"
```

Reflector (deployed via `manifests/infrastructure/controllers/reflector/`) automatically copies this secret to all namespaces.

### Creating BitwardenSecret Resources

Example `BitwardenSecret` for an app:

```yaml
apiVersion: k8s.bitwarden.com/v1
kind: BitwardenSecret
metadata:
  name: radarr-secret
  namespace: media
spec:
  organizationId: "<YOUR_ORG_ID>"
  secretName: radarr-api-key
  authToken:
    secretName: bw-auth-token
    secretKey: token
  map:
    - bwSecretId: "<BITWARDEN_SECRET_ID>"
      secretKeyName: apikey
```

The operator syncs the Bitwarden secret → Kubernetes secret automatically.

---

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

## Manual Creation (without Bitwarden SM Operator)

If you're not using Bitwarden Secrets Manager Operator, create secrets manually:

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

## Bitwarden Secrets Manager Structure

Organize secrets in Bitwarden Secrets Manager:

```
Project: homelab
├── k3s-token
├── tailscale-auth
├── pihole-admin
├── radarr-api-key
├── sonarr-api-key
├── prowlarr-api-key
├── transmission-credentials
└── home-assistant-secrets
```

Each secret's ID is used in the `bwSecretId` field of `BitwardenSecret` resources.
