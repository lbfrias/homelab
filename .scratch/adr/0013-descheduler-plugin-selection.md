# Descheduler Plugin Selection

The descheduler runs as a CronJob every 5 minutes. The `default` profile runs `LowNodeUtilization`, `RemovePodsViolatingInterPodAntiAffinity`, `RemovePodsViolatingNodeAffinity`, and `RemoveDuplicates`. A second `longhorn-csi` profile runs `RemoveDuplicates` scoped to the `longhorn-system` namespace with `evictLocalStoragePods: true`.

`RemoveDuplicates` exists to correct replica stacking after a rolling reboot. When nodes reboot one at a time, any Deployment whose pods are recreated while only a single node is Ready places *all* of its replicas on that node, and Deployments never rebalance on their own. This was observed with Longhorn's CSI sidecars (`csi-attacher`, `csi-provisioner`, `csi-resizer`, `csi-snapshotter`, `longhorn-ui`), which ended up entirely on xialing while peggy sat at 9% memory utilization.

None of the other plugins catch this case:

- `LowNodeUtilization` balances by resource **requests**. Longhorn's CSI sidecars declare none (`resources: {}`), so evicting them does not reduce the source node's utilization and the plugin has no reason to pick them.
- `RemovePodsViolatingInterPodAntiAffinity` only honours `requiredDuringSchedulingIgnoredDuringExecution`. Longhorn's CSI sidecars use a `preferred` anti-affinity rule with weight 1, which is a scheduler hint rather than a violation.

The separate profile is needed because the CSI sidecars mount their gRPC socket directory (`/var/lib/kubelet/plugins/driver.longhorn.io`) as a `hostPath`. The DefaultEvictor counts any `hostPath` or `emptyDir` volume as local storage and filters such pods out entirely, so `RemoveDuplicates` never even saw them: `"pod has local storage and is protected against eviction"`.

## Considered Options

1. **Tune `LowNodeUtilization` thresholds** — peggy sits exactly at the `cpu: 20` threshold and so is classified "appropriately utilized" rather than underutilized. Loosening the thresholds would not help: xialing's memory is dominated by the observability stack (`prometheus-0`, `loki-0`, `alertmanager-0`, `grafana`, `kube-state-metrics`), all hard-pinned via `nodeSelector: kubernetes.io/hostname: xialing`. The descheduler will not evict pods that cannot be rescheduled elsewhere, so the only movable candidates are Longhorn-backed apps whose churn costs data locality for no gain.
2. **Make the CSI anti-affinity `required`** — the Longhorn chart does not expose the sidecar affinity as a value, so this would need a Kustomize patch over Helm-rendered output that has to be re-verified on every chart bump.
3. **Add `topologySpreadConstraints` to the CSI Deployments** — same patching problem as option 2, plus it only addresses Longhorn rather than the general failure mode.
4. **Set `evictLocalStoragePods: true` on the default profile** — one line, but it applies to every plugin in the profile. It would expose `jellyfin` (hostPath `/dev/dri` for hardware transcoding plus NFS media mounts, and no `nodeSelector` pinning it to xialing) and `home-assistant` to eviction by `LowNodeUtilization`, which could reschedule them onto a Raspberry Pi where those host paths do not exist.
5. **`RemoveDuplicates` on the default profile, plus a second profile scoped to `longhorn-system` that permits local-storage eviction** — the default profile keeps its conservative evictor and handles stacked replicas that have no local storage, while the relaxed rule reaches only the CSI sidecars.

We chose option 5, and deliberately left the `LowNodeUtilization` thresholds alone: their current "do nothing" behaviour is correct, because xialing's imbalance is by design.

## Consequences

- Same-owner replicas converge toward an even spread across nodes within a few CronJob runs after a reboot.
- Rolling reboots are safe. `RemoveDuplicates` only considers Ready nodes, skips eviction entirely when fewer than two feasible nodes exist, and evicts only down to `ceil(replicas / feasible nodes)`. With a single node rebooting it moves one pod; with two down it does nothing.
- Node feasibility accounts for taints, node selectors, and node affinity, so pods pinned to a node (such as the observability stack) are never considered duplicates to move.
- Relaxing local-storage protection is confined to `longhorn-system`. Pods elsewhere that use `hostPath` or `emptyDir` — `dnsdist`, `technitium`, `jellyfin`, `home-assistant`, the Flux controllers — stay protected under the default profile's evictor.
- Longhorn's DaemonSets (`longhorn-manager`, `longhorn-csi-plugin`, `engine-image`) are unaffected: `evictDaemonSetPods` defaults to false. `instance-manager` pods each have a distinct owner, so they are never duplicates, and their PDBs allow zero disruptions.
- Single-replica workloads and distinct controllers are unaffected; only genuinely stacked replicas are touched.
- Evictions go through the Eviction API, so PodDisruptionBudgets are respected.
- Every plugin listed under `plugins` must also have a `pluginConfig` entry. Overriding `deschedulerPolicy` replaces the chart's defaults, and a missing entry makes the descheduler fail to start rather than skip the plugin.
