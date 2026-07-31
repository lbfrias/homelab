# Restore Guide

How to recover the homelab from a complete disaster (all nodes wiped).

## Prerequisites

- Longhorn backups exist on NAS (4TB HDD)
- NAS data survived (USB-attached, not wiped with OS)
- This repo is available

## Recovery Steps

### 1. Provision Nodes

Flash OS images and run Ansible:

```bash
# Option A: PXE boot (if configured)
# Start PXE server on desktop, boot nodes from network

# Option B: Manual flash
# - peggy/yelena: Raspberry Pi Imager → Raspberry Pi OS Lite (64-bit)
# - xialing: Fedora Media Writer → Rocky Linux 9

# Run Ansible
cd ansible
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

### 2. Bootstrap K3s

Ansible handles this, but if manual:

```bash
# On xialing (first node)
curl -sfL https://get.k3s.io | K3S_TOKEN=<token> sh -s - server \
  --cluster-init --write-kubeconfig-mode 644 \
  --disable servicelb --disable traefik

# On peggy and yelena
curl -sfL https://get.k3s.io | K3S_TOKEN=<token> sh -s - server \
  --server https://xialing:6443 \
  --disable servicelb --disable traefik
```

> Note: servicelb and traefik disabled because services use macvlan or NodePort.

### 3. Install Core Infrastructure

```bash
# Get kubeconfig
scp xialing:/etc/rancher/k3s/k3s.yaml ~/.kube/config
sed -i 's/127.0.0.1/xialing/g' ~/.kube/config

# Bootstrap Flux
flux bootstrap github \
  --owner=lbfrias \
  --repository=homelab \
  --path=manifests/clusters/homelab
```

Flux will install (in order):
1. Multus + Whereabouts CNI
2. Longhorn
3. Bitwarden Secrets Manager Operator

### 4. Configure Longhorn Backup Target

Once Longhorn is running:

1. Open Longhorn UI: `http://<any-node>:30080` (or via ingress)
2. Go to **Settings** → **Backup Target**
3. Set to: `nfs://<xialing-ip>:/mnt/data/longhorn-backups`
4. Click **Save**

### 5. Restore Volumes

1. Go to **Backup** tab in Longhorn UI
2. Previous backups should appear automatically
3. For each volume to restore:
   - Click the backup
   - Click **Restore**
   - Use original volume name (e.g., `radarr-pvc`)
   - Wait for restore to complete

### 6. Update PVCs

If PVCs were created by Flux before restore, delete and recreate them pointing to restored volumes:

```bash
# Delete existing PVC
kubectl delete pvc radarr-pvc -n media

# Recreate with volumeName pointing to restored volume
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: radarr-pvc
  namespace: media
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  volumeName: radarr-pvc  # Must match restored volume name
  resources:
    requests:
      storage: 2Gi
EOF
```

Alternatively, restore volumes with different names and update the deployments to use the new PVC names.

### 7. Verify Services

```bash
# Check all pods running
kubectl get pods -A

# Check Longhorn volumes healthy
kubectl get volumes -n longhorn-system

# Test services
curl http://radarr.homelab.local
curl http://home-assistant.homelab.local:8123
```

## Partial Recovery

### Single Node Failure

K3s with 3 control plane nodes survives 1 node failure:

1. Reprovision failed node (Ansible)
2. Node auto-rejoins cluster
3. Longhorn rebalances replicas

### Longhorn Volume Corruption

1. Delete corrupted volume
2. Restore from backup (steps 5-6 above)
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
