# Macvlan Networking with Multus

This guide explains how macvlan networking works in this homelab and how to use it in your deployments.

## Overview

By default, Kubernetes pods only have access to the cluster network (Flannel, 10.42.0.0/16). They can reach external services via NAT, but external devices can't initiate connections to pods, and pods can't use L2 protocols like mDNS.

**Multus** solves this by allowing pods to have multiple network interfaces. Combined with **macvlan**, pods can get a "real" IP address on your LAN or VLAN, appearing as a separate device on the network.

```
┌─────────────────────────────────────────────────────────┐
│ Pod                                                     │
│                                                         │
│  eth0 (10.42.x.x)  ←── Flannel (cluster-internal)       │
│  net1 (10.0.0.210) ←── Macvlan (LAN-routable)           │
│                                                         │
└─────────────────────────────────────────────────────────┘
         │                        │
         ▼                        ▼
   K8s Services            Direct L2 access
   Other pods              mDNS, ARP, DHCP
   Egress to internet      Reachable by LAN clients
```

## When to Use Macvlan

Use macvlan when your pod needs:

| Requirement | Example |
|-------------|---------|
| **L2 protocols** | Home Assistant discovering IoT devices via mDNS |
| **Stable LAN IP** | Omada Controller needs a known IP for AP adoption |
| **No SNAT** | DNS server seeing real client IPs for per-client policies |
| **Direct client access** | Services that clients connect to directly (not via ingress) |

**Don't use macvlan** for typical web services — use Ingress or LoadBalancer instead.

## Available NetworkAttachmentDefinitions

NetworkAttachmentDefinitions (NADs) define the macvlan configuration. Different NADs exist because the parent interface name differs between RPi and x86 nodes.

### LAN (10.0.0.0/24)

| NAD Name | Parent Interface | Use On |
|----------|------------------|--------|
| `macvlan-lan` | eth0 | peggy, yelena (RPi) |
| `macvlan-lan-x86` | eno1 | xialing (x86) |

IP range: `10.0.0.200` - `10.0.0.239` (managed by Whereabouts IPAM)

### IoT VLAN (10.0.30.0/24)

| NAD Name | Parent Interface | Use On |
|----------|------------------|--------|
| `macvlan-iot` | eth0.30 | peggy, yelena (RPi) |
| `macvlan-iot-x86` | eno1.30 | xialing (x86) |

IP range: `10.0.30.200` - `10.0.30.239` (managed by Whereabouts IPAM)

## Usage Examples

### Basic: Auto-assigned IP

Add the annotation to request a macvlan interface. Whereabouts will assign an IP from the pool.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-lan
spec:
  containers:
    - name: app
      image: nginx
```

The pod will have two interfaces:
- `eth0` — Flannel (10.42.x.x)
- `net1` — Macvlan (10.0.0.200-239, auto-assigned)

### Specific IP

Request a specific IP address:

```yaml
annotations:
  k8s.v1.cni.cncf.io/networks: |
    [{
      "name": "macvlan-lan",
      "ips": ["10.0.0.210"]
    }]
```

### Multiple Networks

Attach to both LAN and IoT VLAN:

```yaml
annotations:
  k8s.v1.cni.cncf.io/networks: |
    [{
      "name": "macvlan-lan",
      "ips": ["10.0.0.210"]
    },
    {
      "name": "macvlan-iot",
      "ips": ["10.0.30.210"]
    }]
```

### Node Affinity (Important!)

Since NADs are tied to specific parent interfaces, you must ensure pods land on the correct node type:

**For RPi nodes (peggy, yelena):**

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/arch
                operator: In
                values: ["arm64"]
  containers:
    # ...
```

Then use `macvlan-lan` or `macvlan-iot`.

**For x86 node (xialing):**

```yaml
spec:
  nodeSelector:
    kubernetes.io/hostname: xialing
  containers:
    # ...
```

Then use `macvlan-lan-x86` or `macvlan-iot-x86`.

### Complete Deployment Example

Home Assistant on IoT VLAN with a specific IP:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: home-assistant
  namespace: home
spec:
  replicas: 1
  selector:
    matchLabels:
      app: home-assistant
  template:
    metadata:
      labels:
        app: home-assistant
      annotations:
        k8s.v1.cni.cncf.io/networks: |
          [{
            "name": "macvlan-iot",
            "ips": ["10.0.30.210"]
          }]
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/arch
                    operator: In
                    values: ["arm64"]
      containers:
        - name: home-assistant
          image: homeassistant/home-assistant:latest
          # ...
```

## Verifying Macvlan

### Check pod interfaces

```bash
kubectl exec -it <pod> -- ip addr
```

Look for `net1` with an IP in the macvlan range.

### Check IP allocation

```bash
kubectl get ippools.whereabouts.cni.cncf.io -A
kubectl get overlappingrangeipreservations.whereabouts.cni.cncf.io -A
```

### Test connectivity

From the pod:
```bash
kubectl exec -it <pod> -- ping 10.0.0.1  # Gateway
```

From a LAN device:
```bash
ping 10.0.0.210  # Pod's macvlan IP
```

## Troubleshooting

### Pod stuck in ContainerCreating

Check Multus logs:
```bash
kubectl logs -n kube-system -l app=multus --tail=50
```

Common causes:
- Wrong NAD name in annotation
- Pod scheduled on wrong node (interface mismatch)
- IP pool exhausted

### Can't reach pod from LAN

- Verify the pod got the macvlan interface: `kubectl exec <pod> -- ip addr`
- Check the IP is in the expected range
- Ensure no firewall blocking on the pod or network

### IP conflicts

Whereabouts tracks allocations. If you see conflicts:
```bash
# Check current allocations
kubectl get ippools.whereabouts.cni.cncf.io -A -o yaml
```

The IP ranges (200-239) are reserved for Kubernetes — don't assign these IPs to other devices on your network.

## Architecture Details

### Components

| Component | Purpose | Location |
|-----------|---------|----------|
| Multus | CNI multiplexer | DaemonSet in kube-system |
| Whereabouts | IPAM (IP allocation) | Runs as CNI plugin |
| NADs | Network definitions | kube-system namespace |

### How It Works

1. Pod created with `k8s.v1.cni.cncf.io/networks` annotation
2. Kubelet calls Multus (configured as primary CNI)
3. Multus calls Flannel for `eth0`, then the NAD's CNI (macvlan) for `net1`
4. Macvlan plugin creates interface attached to host's physical NIC
5. Whereabouts allocates/reserves IP from the pool
6. Pod starts with both interfaces configured

### File Locations (K3s)

- CNI binaries: `/var/lib/rancher/k3s/data/cni/`
- CNI config: `/var/lib/rancher/k3s/agent/etc/cni/net.d/`
- Whereabouts state: Stored as Kubernetes CRDs (ippools, overlappingrangeipreservations)
