# Step 5.11 — NGINX Ingress Controller & Public Access

**What to build:** NGINX Ingress Controller with macvlan LAN IP for internal service access via `*.frias.app` domain, plus Cloudflare Tunnel for public access.

**Blocked by:** Step 5.4 (Multus/Macvlan), Step 5.5 (DNS for Technitium zone config)

**Status:** done

## Architecture

### Internal Access (LAN)

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

### Public Access (Internet)

```
Internet clients
    │
    ▼ https://home.frias.app
Cloudflare Edge (DNS + Tunnel)
    │ (encrypted tunnel)
    ▼
cloudflared connector (cluster pod)
    │
    ▼ http://home-assistant.home.svc.cluster.local:8123
Home Assistant
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
- [x] Verify: NGINX responds on `10.0.0.30:80` and `:443`

### cert-manager

- [x] Deploy cert-manager in `manifests/infrastructure/controllers/cert-manager/` (v1.21.1)
- [x] Create Cloudflare API token BitwardenSecret
- [x] Create ClusterIssuer `letsencrypt-prod` with DNS-01 solver (Cloudflare)
- [x] Create ClusterIssuer `letsencrypt-staging` for testing
- [x] Add Cloudflare API token to Bitwarden SM
- [x] Set `ACME_EMAIL` in cluster-vars.yaml
- [x] Verify: Certificates issued successfully

### DNS Configuration

- [x] Configure Technitium zone for `frias.app` (override for internal resolution)
- [x] Add wildcard A record: `*.frias.app → 10.0.0.30`
- [x] Verify: Internal DNS resolves `*.frias.app` to NGINX IP

### Sample Ingress (Home Assistant)

- [x] Create Ingress resource for Home Assistant at `home.frias.app`
- [x] Configure TLS with cert-manager annotation
- [x] Add `http.use_x_forwarded_for` and `trusted_proxies` to HA config
- [x] Verify: `https://home.frias.app` works from LAN client

### All App Ingresses

- [x] Home Assistant: `home.frias.app`
- [x] Jellyfin: `watch.frias.app`
- [x] Radarr: `movies.frias.app`
- [x] Sonarr: `tv.frias.app`
- [x] Prowlarr: `indexer.frias.app`
- [x] Bazarr: `subs.frias.app`
- [x] Transmission: `downloads.frias.app`
- [x] Kavita: `read.frias.app`
- [x] Mylar3: `comics.frias.app`
- [x] Longhorn: `longhorn.frias.app`

### Cloudflare Tunnel (Public Access)

- [x] Create namespace `cloudflared` in `manifests/apps/network/cloudflared/`
- [x] Create BitwardenSecret for tunnel token
- [x] Deploy cloudflared connector (2026.7.3) with `--metrics` flag for health probes
- [x] Add `cloudflared` to bw-auth-token reflection-allowed-namespaces
- [x] Configure public hostname in Cloudflare dashboard:
  - Hostname: `home.frias.app`
  - Service: `http://home-assistant.home.svc.cluster.local:8123`
- [x] Verify: `https://home.frias.app` accessible from internet

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

manifests/apps/network/cloudflared/
├── kustomization.yaml
├── namespace.yaml
├── bitwardensecret.yaml  # Tunnel token from Bitwarden
└── deployment.yaml       # cloudflared connector

manifests/apps/home/home-assistant/
└── ingress.yaml          # home.frias.app

manifests/apps/media/*/
└── ingress.yaml          # Per-app ingresses

manifests/infrastructure/configs/longhorn/
└── ingress.yaml          # longhorn.frias.app
```

## Bitwarden Secrets

| Name | UUID | Purpose |
|------|------|---------|
| `cloudflare-dns-frias-app` | `5e35e756-70e5-40f8-8a54-b49b00167290` | Cloudflare API Token (Zone:DNS:Edit) |
| `cloudflare-tunnel-token` | `fd5cb98a-8112-48ce-bf6a-b49b002be284` | Cloudflare Tunnel token |

## Verification

- [x] NGINX Ingress pod running with macvlan IP `10.0.0.30`
- [x] cert-manager issues certificates successfully
- [x] `https://home.frias.app` accessible with valid TLS from LAN
- [x] Certificate shows Let's Encrypt issuer
- [x] Cloudflare Tunnel healthy (4 connections to edge)
- [x] `https://home.frias.app` accessible from internet

## Home Assistant Configuration

Required config additions for reverse proxy support:

```yaml
homeassistant:
  external_url: "https://home.frias.app"
  internal_url: "https://home.frias.app"

http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 10.42.0.0/16  # K3s pod network

my:        # For "My Home Assistant" redirect buttons
zeroconf:  # For mDNS discovery
```

## Notes

- Cloudflared needs `--metrics 0.0.0.0:2000` flag to expose health endpoint for liveness/readiness probes
- bw-auth-token must include `cloudflared` in `reflection-allowed-namespaces` annotation
- Cloudflare Tunnel public hostname configured via dashboard: Networking → Tunnels → homelab → Routes
