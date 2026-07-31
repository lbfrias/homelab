# Restore Guide

How to recover the homelab from a complete disaster (all nodes wiped).

## Prerequisites

- Longhorn backups exist on NAS (4TB HDD)
- NAS data survived (USB-attached, not wiped with OS)
- This repo is available

## Recovery Steps

### 1. Provision Nodes

Flash OS images (see [CONTEXT.md](../CONTEXT.md#hardware) for OS per node), then run Ansible:

```bash
cd ansible
ansible-playbook -e @vars.yaml -e @secrets.local.yaml site.yaml
```

This runs Steps 2-4 (bootstrap, K3s, Flux) in sequence. Kubeconfig is saved locally automatically.

Flux will install (in order):
1. Multus + Whereabouts CNI
2. Longhorn (with backup target pre-configured via GitOps)
3. Bitwarden Secrets Manager Operator

### 2. Restore Volumes

Access the Longhorn UI:

```bash
kubectl port-forward svc/longhorn-frontend -n longhorn-system 8080:80
# Open http://localhost:8080
```

Restore volumes from backup:

1. Go to **Backup** tab — previous backups should appear automatically
2. For each volume to restore:
   - Click the backup
   - Click **Restore**
   - Check **Create PVC** and use original name (e.g., `radarr-pvc`)
   - Wait for restore to complete

### 3. Verify Services

```bash
# Check all pods running
kubectl get pods -A

# Check Longhorn volumes healthy
kubectl get volumes.longhorn.io -n longhorn-system

# Spot-check a service via port-forward
kubectl port-forward svc/radarr-svc -n media 7878:7878
# Open http://localhost:7878

kubectl port-forward svc/home-assistant -n home 8123:8123
# Open http://localhost:8123
```

## Partial Recovery

### Single Node Failure

K3s with 3 control plane nodes survives 1 node failure:

1. Reprovision failed node (Ansible)
2. Node auto-rejoins cluster
3. Longhorn rebalances replicas

### Longhorn Volume Corruption

1. Delete corrupted volume
2. Restore from backup (step 2 above)
3. Restart affected deployment

### Lost NAS Data

If 4TB HDD fails, Longhorn backups are lost. Recovery:
- Redeploy apps from scratch
- Restore app-native backups if available (radarr/sonarr exports)
- Media files must be re-acquired

## Testing Recovery

Periodically test restore capability:

```bash
# Test: Restore prowlarr (non-critical) to verify backup integrity
# 1. Delete prowlarr PVC
# 2. Restore from Longhorn backup
# 3. Verify prowlarr works with restored data
# 4. Document any issues
```

## Test Log

| Date | Scope | Result |
|------|-------|--------|
| 2026-08-01 | Full media stack restored via Longhorn | ✓ Success |
