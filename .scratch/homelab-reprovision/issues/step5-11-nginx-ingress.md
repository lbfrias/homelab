# Step 5.11 — NGINX Ingress Controller

**What to build:** NGINX Ingress Controller with macvlan LAN IP for internal service access via `*.frias.app` domain.

**Blocked by:** Step 5.4 (Multus/Macvlan), Step 5.5 (DNS for Technitium zone config)

**Status:** in-progress

## Architecture

```
Clients (browser, apps)
    │
    ▼ https://jellyfin.frias.app
Technitium (split DNS: *.frias.app → 10.0.0.30)
    │
    ▼
NGINX Ingress (macvlan: 10.0.0.30)
    │
    ▼ proxy to ClusterIP services
Jellyfin, Radarr, Sonarr, etc.
```

## Decisions

| Decision | Value |
|----------|-------|
| Ingress controller | NGINX Ingress Controller |
| IP assignment | Macvlan via `macvlan-lan` NAD with static IP |
| IP address | `10.0.0.30` |
| Domain | `*.frias.app` |
| TLS | cert-manager + Let's Encrypt (DNS-01 via Cloudflare) |
| DNS | Split DNS (Cloudflare public + Technitium internal) |

## Tasks

### NGINX Ingress Controller

- [x] Create namespace `ingress-nginx` in `manifests/infrastructure/controllers/nginx-ingress/`
- [x] Deploy NGINX Ingress Controller via HelmRelease (v4.15.1)
- [x] Configure macvlan annotation using `macvlan-lan` NAD with static IP `10.0.0.30`
- [x] Create IngressClass `nginx` as default
- [ ] Verify: NGINX responds on `10.0.0.30:80` and `:443`

### cert-manager

- [x] Deploy cert-manager in `manifests/infrastructure/controllers/cert-manager/` (v1.21.1)
- [x] Create Cloudflare API token BitwardenSecret
- [x] Create ClusterIssuer `letsencrypt-prod` with DNS-01 solver (Cloudflare)
- [x] Create ClusterIssuer `letsencrypt-staging` for testing
- [ ] TODO: Add Cloudflare API token to Bitwarden SM, get secret ID
- [x] Set `ACME_EMAIL` in cluster-vars.yaml
- [ ] Verify: Can issue test certificate for `test.frias.app`

### DNS Configuration

- [x] Configure Technitium zone for `frias.app` (override for internal resolution)
- [x] Add wildcard A record: `*.frias.app → 10.0.0.30`
- [ ] Verify: `dig jellyfin.frias.app @10.0.0.95` returns `10.0.0.30`

### Sample Ingress (Home Assistant)

- [x] Create Ingress resource for Home Assistant at `hass.frias.app`
- [x] Configure TLS with cert-manager annotation
- [ ] Verify: `https://hass.frias.app` works from LAN client

### IP Plan Updates

- [x] Update `docs/ip-plan.yaml` — add NGINX IP `10.0.0.30` in `infra_pods` section
- [x] Update `manifests/infrastructure/controllers/cluster-vars.yaml` — add `NGINX_INGRESS_IP`

## Files Created

```
manifests/infrastructure/controllers/nginx-ingress/
├── kustomization.yaml
├── namespace.yaml
└── release.yaml          # HelmRelease with macvlan annotation

manifests/infrastructure/controllers/cert-manager/
├── kustomization.yaml
├── namespace.yaml
└── release.yaml          # HelmRelease

manifests/infrastructure/configs/cert-manager/
├── kustomization.yaml
├── bitwardensecret.yaml  # Cloudflare API token
└── clusterissuers.yaml   # staging + prod issuers
```

## Verification

- [ ] NGINX Ingress pod running with macvlan IP `10.0.0.30`
- [ ] cert-manager issues certificates successfully
- [ ] `https://jellyfin.frias.app` accessible with valid TLS from LAN
- [ ] Certificate shows Let's Encrypt issuer

## Before Deploying

1. Create Cloudflare API token with Zone:DNS:Edit for `frias.app`
2. Add token to Bitwarden Secrets Manager
3. Update `CLOUDFLARE_API_TOKEN_SECRET_ID` in `configs/cert-manager/bitwardensecret.yaml`
4. Set `ACME_EMAIL` in `cluster-vars.yaml`
