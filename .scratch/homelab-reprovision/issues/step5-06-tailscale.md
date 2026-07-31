# Step 5.6 — Tailscale Subnet Router

**What to build:** Tailscale subnet router for remote access to entire home network.

**Blocked by:** Step 5.3 (Bitwarden Secrets)

**Status:** done

## Design Decision

Using a simple **subnet router deployment** instead of Tailscale Operator:
- Operator is for exposing K8s Services to tailnet
- Subnet router advertises LAN subnets (10.0.0.0/24, 10.0.30.0/24)
- No `hostNetwork: true` needed — userspace networking handles routing

## Tasks

- [x] Create Tailscale subnet router in `manifests/apps/network/tailscale/`
- [x] Create BitwardenSecret for Tailscale auth key
- [x] Add Bitwarden secret IDs (org ID + auth key secret ID) to bitwardensecret.yaml
- [x] Enable apps kustomization in `manifests/kustomization.yaml`
- [x] Create Flux Kustomization for apps in `manifests/infrastructure/kustomizations.yaml`
- [x] Deploy and verify: Approve routes in Tailscale admin, test remote access

## Resolution

Tailscale subnet router deployed and operational (2026-08-01).

## Files Created

```
manifests/apps/network/
├── kustomization.yaml
└── tailscale/
    ├── namespace.yaml
    ├── rbac.yaml
    ├── bitwardensecret.yaml
    ├── deployment.yaml
    └── kustomization.yaml
```

## Manual Steps Required

1. Create Tailscale auth key in admin console (reusable, ephemeral recommended)
2. Store auth key in Bitwarden Secrets Manager
3. Update `bitwardensecret.yaml` with your Bitwarden org ID and secret ID
4. After deploy: Approve subnet routes in Tailscale admin console
