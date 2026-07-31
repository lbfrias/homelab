# Step 5.9 — Media Stack

**What to build:** Jellyfin (with hardware transcoding), Radarr, Sonarr, Prowlarr, Bazarr, Transmission, Kavita, Mylar3.

**Blocked by:** Step 5.2 (Longhorn)

**Status:** done

## Tasks

- [x] Create media namespace in `manifests/apps/media/`
- [x] Migrate manifests from old `homelab-ansible` repo
- [x] Configure shared NFS mount for media files
- [x] Create individual PVCs for each app's config
- [x] Jellyfin: GPU passthrough for Intel Quick Sync, node selector for xialing
- [x] Configure service access (NodePort or macvlan)
- [x] Verify: Jellyfin streams with hardware transcoding
- [x] Verify: *arr apps can manage libraries

## Resolution

All volumes restored via Longhorn. Media stack fully operational (2026-08-01).
