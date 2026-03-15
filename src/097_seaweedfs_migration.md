# SeaweedFS: Replacing Longhorn, Third Time's the Charm

## The Backdrop

SeaweedFS and I have history. Deployed it on USB SSDs ([chapter 63](./063_seaweedfs.md)), tore it down, put it back on USB SSDs via the operator ([chapter 79](./079_seaweedfs.md)), watched it die when someone bumped the USB-to-SATA cables (physically, not metaphorically), deployed Longhorn ([chapter 92](./092_longhorn.md)), and now here we are. Going back to SeaweedFS. On the same NVMe drives. Using the same mount path.

The difference this time: NVMe HATs. No more USB-to-SATA cables to bump. The drives are bolted directly to the Pi 5 boards. You'd have to physically disassemble a node to disconnect one. That's the kind of reliability guarantee I can respect.

Why leave Longhorn? It worked fine, honestly. 30 pods across the cluster, decent performance, web UI for when I wanted to stare at disk utilization bars. But it was heavier than it needed to be — DaemonSets on every node, iSCSI plumbing, an engine image DaemonSet, instance managers spawning per-node, CSI plugins everywhere. And the volumes were physically constrained to the 4 NVMe nodes, which meant things like Prometheus and Alertmanager needed `nodeSelector` configs to land on NVMe nodes. SeaweedFS with a CSI driver lets any pod on any node mount a volume — the CSI mount DaemonSet runs everywhere and talks to the filer over the network.

The plan: deploy SeaweedFS server (master + volume + filer), deploy the CSI driver, migrate every Longhorn consumer, then nuke Longhorn from orbit.

## The Architecture

SeaweedFS this time is simpler than my previous deployments. No operator, no S3 API, no workers. Just the Helm chart:

- **1 Master**: Cluster coordinator. Handles volume allocation, topology. Runs on an NVMe node.
- **4 Volume Servers**: One per Pi 5 NVMe node. Stores actual data on `/var/lib/longhorn` (yes, still called that — the Talos `userVolumes` config names the NVMe mount and I'm not renaming it). Replication set to `001` — one extra copy on a different volume server.
- **1 Filer**: Filesystem abstraction over the volume servers. Uses LevelDB2 for metadata. Runs on an NVMe node.

For the CSI driver:
- **1 Controller**: Handles CreateVolume/DeleteVolume.
- **14 Node DaemonSet pods**: Run on every worker node. Handle NodePublishVolume.
- **14 Mount DaemonSet pods**: Run on every worker node. Provide the FUSE mount service so volumes can be mounted anywhere in the cluster.

Total pod count: 6 server + ~29 CSI = ~35 pods. Longhorn was 30 pods. Slightly more, but the SeaweedFS CSI pods are tiny and the tradeoff is that workloads aren't pinned to NVMe nodes anymore.

## Deploying the Server

Standard Flux CD structure — six files in `gitops/infrastructure/seaweedfs/`:

```
seaweedfs/
├── kustomization.yaml
├── namespace.yaml          # seaweedfs, privileged PSA (for hostPath)
├── server-repository.yaml  # HelmRepository → seaweedfs.github.io/seaweedfs/helm
├── server-release.yaml     # HelmRelease for master/volume/filer
├── csi-repository.yaml     # HelmRepository → seaweedfs.github.io/seaweedfs-csi-driver/helm
└── csi-release.yaml        # HelmRelease for CSI driver
```

Key server values:

```yaml
global:
  enableReplication: true
  replicationPlacement: "001"

master:
  replicas: 1
  data:
    type: "hostPath"
    hostPathPrefix: /var/lib/longhorn
  nodeSelector: |
    node.kubernetes.io/disk-type: nvme
  affinity: ""   # allow co-location

volume:
  replicas: 4
  dataDirs:
    - name: data1
      type: "hostPath"
      hostPathPrefix: /var/lib/longhorn
      maxVolumes: 0
  nodeSelector: |
    node.kubernetes.io/disk-type: nvme

filer:
  replicas: 1
  data:
    type: "hostPath"
    hostPathPrefix: /var/lib/longhorn
  nodeSelector: |
    node.kubernetes.io/disk-type: nvme
  affinity: ""
```

The `affinity: ""` on master and filer overrides the chart's default anti-affinity, letting them co-locate on the same NVMe node. With only 4 NVMe nodes and 6 server pods, some sharing is inevitable.

Committed, pushed, waited for Flux. All six server pods came up without drama.

```
$ kubectl get pods -n seaweedfs
NAME                  READY   STATUS    RESTARTS   AGE
seaweedfs-filer-0     2/2     Running   0          3m
seaweedfs-master-0    1/1     Running   0          3m
seaweedfs-volume-0    1/1     Running   0          3m
seaweedfs-volume-1    1/1     Running   0          3m
seaweedfs-volume-2    1/1     Running   0          3m
seaweedfs-volume-3    1/1     Running   0          3m
```

## The CSI Driver: A Two-Bug Symphony

The CSI driver was less cooperative.

### Bug 1: The Phantom Flag

The CSI node DaemonSet immediately crashed:

```
flag provided but not defined: -mountEndpoint
```

Turns out the chart (v0.2.11) generates a `--mountEndpoint` flag for the new mount service feature, but the `latest` Docker image tag doesn't actually include it. The `latest` tag on Docker Hub is stale — it's some ancient build that predates the mount service feature entirely. Classic.

### Bug 2: The Missing Node ID

The CSI controller also refused to start:

```
Precondition failed: driver requires node id to be set, use -nodeid=
```

The chart doesn't pass `--nodeid` to the controller pod. This might be fine for newer versions of the binary, but the `latest` image was too old to handle it gracefully.

### The Fix

Both bugs had the same root cause: the `latest` Docker tag was garbage. Pinning all three images (controller, node plugin, mount service) to `v1.4.5`:

```yaml
csiDriverImages:
  controller:
    repository: chrislusf/seaweedfs-csi-driver
    tag: "v1.4.5"
  node:
    repository: chrislusf/seaweedfs-csi-driver
    tag: "v1.4.5"
  mount:
    repository: chrislusf/seaweedfs-csi-driver
    tag: "v1.4.5"
```

Fixed both issues. The v1.4.5 binary supports `--mountEndpoint` and handles the missing nodeid for controller mode.

### Bug 3: The DNS Trap

After the images were fixed, the CSI driver still couldn't create volumes. The filer address was wrong:

```yaml
seaweedfsFiler: "seaweedfs-filer.seaweedfs.svc.cluster.local:8888"
```

But the actual service name is:

```yaml
seaweedfsFiler: "seaweedfs-seaweedfs-filer.seaweedfs.svc.cluster.local:8888"
```

The Helm chart prefixes service names with `{release-name}-{chart-name}-{component}`. Since both the release and chart are named "seaweedfs," you get `seaweedfs-seaweedfs-filer`. Beautiful.

After that fix, the CSI driver was fully operational. Created a test PVC, wrote data, read it back. SeaweedFS works.

## The Migration

Thirteen PVCs to migrate, across four namespaces. All using `storageClassName: longhorn`, all needing to switch to `storageClassName: seaweedfs`.

### The Easy Ones: Gatus

One PVC, one Deployment. Changed the storageClass in `gitops/apps/gatus/pvc.yaml`, deleted the old PVC, let Flux recreate it. Done.

### The StatefulSet Conundrum: Garage

Garage has 8 PVCs (4 data + 4 meta) attached via VolumeClaimTemplates on a StatefulSet. Kubernetes doesn't let you modify VolumeClaimTemplates on existing StatefulSets. So:

1. Update storageClass in `gitops/infrastructure/garage/statefulset.yaml`
2. Delete all 8 PVCs
3. Delete the StatefulSet
4. Let Flux recreate everything fresh

All 8 PVCs came back on SeaweedFS. Garage spent a while rebuilding its metadata ring from scratch (lots of restarts, nodes rediscovering each other), but that was expected.

### Prometheus and Alertmanager

The kube-prometheus-stack operator manages its own StatefulSets. Changed `storageClassName` in the HelmRelease values, plus removed the `nodeSelector` constraints — no longer needed since SeaweedFS CSI serves volumes to any node.

```yaml
# Before
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: longhorn
    nodeSelector:
      node.kubernetes.io/disk-type: nvme

# After
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: seaweedfs
    # no nodeSelector — any node can mount SeaweedFS volumes
```

Deleted the StatefulSets and PVCs, operator recreated them. Both came up on SeaweedFS.

### Loki and Tempo: The Helm Release Hell

This is where things got interesting. "Interesting" as in "I spent an hour fighting Flux's Helm controller."

Loki and Tempo both use StatefulSets with VolumeClaimTemplates. Updated the storageClass in both HelmRelease values. But Helm tried to do an upgrade, which hit the immutable VolumeClaimTemplate wall:

```
Helm upgrade failed: cannot patch "monitoring-loki" with kind StatefulSet:
StatefulSet.apps "monitoring-loki" is invalid: spec: Forbidden: updates to
statefulset spec for fields other than 'replicas', 'ordinals', 'template',
'updateStrategy', 'revisionHistoryLimit', 'persistentVolumeClaimRetentionPolicy'
and 'minReadySeconds' are forbidden
```

OK, delete the StatefulSets so the upgrade can recreate them. But now the Helm rollback mechanism kicked in — upgrade failed, so Flux tried to rollback. The rollback failed because the StatefulSet no longer existed:

```
Helm rollback failed: statefulsets.apps "monitoring-loki" not found
```

Now we're in a loop: upgrade fails (VCT immutable) → rollback fails (StatefulSet missing) → repeat forever.

I tried:
- Deleting the HelmRelease CRs (Flux recreated them from Git, but the Helm release secrets persisted)
- Suspending and resuming (same loop)
- `helm uninstall` (release not found — because `helm list` showed nothing)

The mystery: `helm list -n monitoring -a` showed zero releases, but Flux was clearly finding Helm state somewhere. Turns out Flux stores its Helm release secrets in the **`flux-system` namespace**, not the target namespace. That's why `helm list` showed nothing — it was looking in the wrong namespace.

```
$ kubectl get secrets -n flux-system -l owner=helm | grep loki
sh.helm.release.v1.monitoring-loki.v66    helm.sh/release.v1   1   30h
sh.helm.release.v1.monitoring-loki.v68    helm.sh/release.v1   1   9m
sh.helm.release.v1.monitoring-loki.v69    helm.sh/release.v1   1   7m
sh.helm.release.v1.monitoring-loki.v70    helm.sh/release.v1   1   2m
sh.helm.release.v1.monitoring-loki.v71    helm.sh/release.v1   1   2m
```

There's the bastard. v66 was the original, and v68-v71 were failed upgrade/rollback attempts piling up.

The fix:

```bash
# 1. Suspend HelmReleases
kubectl patch hr loki -n flux-system --type merge -p '{"spec":{"suspend":true}}'
kubectl patch hr tempo -n flux-system --type merge -p '{"spec":{"suspend":true}}'

# 2. Delete ALL Helm release secrets
kubectl delete secrets -n flux-system -l name=monitoring-loki
kubectl delete secrets -n flux-system -l name=monitoring-tempo

# 3. Delete StatefulSets and old PVCs
kubectl delete sts monitoring-loki monitoring-tempo -n monitoring
kubectl delete pvc storage-monitoring-loki-0 storage-monitoring-tempo-0 -n monitoring --force

# 4. Resume — Flux does a fresh install
kubectl patch hr loki -n flux-system --type merge -p '{"spec":{"suspend":false}}'
kubectl patch hr tempo -n flux-system --type merge -p '{"spec":{"suspend":false}}'
```

Both came up with fresh installs on SeaweedFS PVCs:

```
NAME                    READY   STATUS
loki    True    Helm install succeeded for release monitoring/monitoring-loki.v1
tempo   True    Helm install succeeded for release monitoring/monitoring-tempo.v1
```

Note the v1 — clean slate.

## Removing Longhorn

With zero Longhorn PVCs remaining:

```
$ kubectl get pvc -A | grep longhorn
<nothing>
```

The removal was straightforward:

1. Removed `longhorn` from `gitops/infrastructure/kustomization.yaml`
2. Deleted `gitops/infrastructure/longhorn/` directory (4 files)
3. Committed and pushed
4. Flux pruned the HelmRelease automatically (pruning enabled on the infrastructure kustomization)
5. Cleaned up Helm release secrets from flux-system namespace
6. Deleted all 22 Longhorn CRDs manually
7. Removed the validating/mutating webhook configurations
8. Cleared finalizers on stuck EngineImage and Node CRD resources

The `longhorn-system` namespace got stuck in Terminating state because the admission webhook was gone but CRD resources still had `longhorn.io` finalizers:

```
Failed to delete all resource types, 1 remaining: Internal error occurred:
failed calling webhook "validator.longhorn.io": service
"longhorn-admission-webhook" not found
```

Classic chicken-and-egg — the webhook service was deleted but K8s was still trying to validate resource deletions through it. Deleting the webhook configurations and patching out the finalizers unstuck it.

## Final State

All 13 PVCs across the cluster are now on SeaweedFS:

| Namespace | PVC | Size | Purpose |
|-----------|-----|------|---------|
| garage | data-garage-{0..3} | 100Gi x4 | S3 object storage data |
| garage | meta-garage-{0..3} | 1Gi x4 | Garage metadata |
| gatus | gatus-data | 1Gi | Status page data |
| monitoring | prometheus-...-0 | 16Gi | Prometheus TSDB |
| monitoring | alertmanager-...-0 | 1Gi | Alertmanager state |
| monitoring | storage-monitoring-loki-0 | 16Gi | Log storage |
| monitoring | storage-monitoring-tempo-0 | 10Gi | Trace storage |

StorageClasses remaining: `seaweedfs`, `local-path`, `local-path-usb`. No more `longhorn`.

SeaweedFS server: 6 pods. CSI driver: ~29 pods (mostly DaemonSets). Nothing is constrained to NVMe nodes anymore — any pod on any of the 14 worker nodes can mount a SeaweedFS volume. The actual data lives on the 4 NVMe drives with `001` replication, so everything has a copy on a different volume server.

Third time deploying SeaweedFS on this cluster. This time the drives are physically bolted to the boards. I'm cautiously optimistic. Famous last words.
