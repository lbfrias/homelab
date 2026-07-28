# Step 5.9 — Media Stack

**What to build:** Jellyfin (with hardware transcoding), Radarr, Sonarr, Prowlarr, Bazarr, Transmission, Kavita, Mylar3.

**Blocked by:** Step 5.1 (MetalLB), Step 5.2 (Longhorn)

**Status:** ready

## Tasks

- [ ] Create media namespace in `manifests/apps/media/`
- [ ] Migrate manifests from old `homelab-ansible` repo
- [ ] Configure shared NFS mount for media files
- [ ] Create individual PVCs for each app's config
- [ ] Jellyfin: GPU passthrough for Intel Quick Sync, node selector for xialing
- [ ] Configure LoadBalancer services
- [ ] Verify: Jellyfin streams with hardware transcoding
- [ ] Verify: *arr apps can manage libraries
