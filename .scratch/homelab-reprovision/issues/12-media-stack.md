# 12 — Media stack (Jellyfin + *arr apps)

**What to build:** Full media stack deployed: Jellyfin (with hardware transcoding), Radarr, Sonarr, Prowlarr, Bazarr, Transmission, Kavita, Mylar3.

**Blocked by:** 04 — MetalLB, 05 — Longhorn

**Status:** ready-for-agent

- [ ] Create media namespace and deployments in `manifests/apps/media/`
- [ ] Migrate manifests from old `homelab-ansible` repo, adapting as needed
- [ ] Configure shared NFS mount for media files (from xialing 4TB HDD)
- [ ] Create individual PVCs for each app's config
- [ ] Jellyfin: Add GPU passthrough for Intel Quick Sync, node selector for xialing
- [ ] Configure LoadBalancer services for each app
- [ ] Verify: Jellyfin can stream with hardware transcoding
- [ ] Verify: *arr apps can manage libraries
