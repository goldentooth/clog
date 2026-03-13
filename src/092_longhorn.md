# Longhorn: Distributed Storage on NVMe

The whole point of those Pi 5 nodes was storage. Four 932GB NVMe SSDs sitting in the cluster, waiting for a distributed storage system to give them purpose. The previous occupant of that role, SeaweedFS, was torn down a few days ago after a long slide into irrelevance — it was an object store bolted onto USB SSDs with an operator that fought me at every turn. The docker registry it backed had been in CrashLoopBackOff since the teardown. Time for something better.

## Why Longhorn

Longhorn is a distributed block storage system for Kubernetes. Unlike SeaweedFS (object storage, S3 API, filer abstraction, operator complexity), Longhorn provides plain old PersistentVolumes. You create a PVC, Longhorn allocates space on the NVMe drives, replicates the data across nodes, and serves it over iSCSI. No S3 buckets, no filer processes, no operator CRDs that refuse to delete cleanly. Just block storage.

The architecture:

- **Longhorn Manager**: DaemonSet on every worker node. Creates node resources, manages replicas, orchestrates volume operations.
- **Engine + Instance Manager**: Per-node processes that handle the actual iSCSI target serving and data replication.
- **CSI Plugin**: DaemonSet on every node. Handles volume attach/detach/mount so any pod on any node can consume Longhorn volumes.
- **UI**: Optional web dashboard. Handy for seeing disk utilization at a glance.

Data lives on the 4 Pi 5 NVMe nodes. Any pod on any worker node can mount a Longhorn volume — the CSI plugin attaches it over iSCSI from whichever storage node holds the replica.

## The Prerequisites

The Pi 5 nodes were already mostly ready from the Ubuntu setup work:

- **NVMe SSDs**: 932GB each, formatted XFS, mounted at `/var/lib/longhorn`
- **open-iscsi**: Installed, `iscsid` enabled and running
- **nfs-common**: Installed (for RWX support later)

One thing missing: the `iscsi_tcp` kernel module wasn't set to load at boot on three of the four nodes (manderly had it from testing). Quick fix:

```bash
for ip in 10.4.0.22 10.4.0.23 10.4.0.24 10.4.0.25; do
  ssh nathan@$ip 'echo iscsi_tcp | sudo tee /etc/modules-load.d/iscsi.conf'
done
```

The Talos worker nodes (Pi 4B) already had the `iscsi-tools` and `util-linux-tools` extensions configured in `talconfig.yaml`, plus a `/var/lib/longhorn` bind mount in the kubelet extra mounts. Past me actually planned ahead for once.

## The GitOps Manifests

Following the existing Flux CD pattern — same structure as Cilium, Prometheus, everything else:

```
gitops/infrastructure/longhorn/
├── kustomization.yaml
├── namespace.yaml      # longhorn-system, privileged PSA
├── repository.yaml     # HelmRepository → charts.longhorn.io
└── release.yaml        # HelmRelease with all the values
```

Added `- longhorn` to the infrastructure kustomization, committed, pushed, waited for Flux.

### Key Helm Values

The important decisions:

```yaml
defaultSettings:
  createDefaultDiskLabeledNodes: true
  defaultDataPath: /var/lib/longhorn
  defaultReplicaCount: 2
  storageMinimalAvailablePercentage: 15
  nodeDownPodDeletionPolicy: delete-both-statefulset-and-deployment-pod

persistence:
  defaultClassReplicaCount: 2
  defaultClass: true
```

**`createDefaultDiskLabeledNodes: true`** is the key one. Longhorn only creates storage disks on nodes that have the label `node.longhorn.io/create-default-disk=true`. This means the 12 Pi 4B nodes and the GPU node don't accidentally become storage backends — only the 4 Pi 5 nodes with their NVMe drives.

**Replica count 2**: With 4 storage nodes, 3 replicas would mean 75% of nodes hold every volume. That's a lot of cross-node replication traffic for marginal benefit. 2 replicas survives a single node failure, which is the realistic failure mode for a home cluster.

**Default StorageClass**: Longhorn becomes the cluster default. New PVCs that don't specify a class get Longhorn automatically. The old `local-path` (SD card) and `local-path-usb` classes still exist for workloads that want them.

## The Three Mistakes

### Mistake 1: systemManagedComponentsNodeSelector

My first attempt restricted Longhorn system components to NVMe nodes only:

```yaml
defaultSettings:
  systemManagedComponentsNodeSelector: "node.kubernetes.io/disk-type:nvme"
```

This seemed logical — keep Longhorn's manager, engine images, and CSI plugin on the storage nodes. The problem: **the CSI plugin must run on every node**, not just storage nodes. Without the CSI plugin, a node can't mount Longhorn volumes. A pod scheduled on `inchfield` (a Pi 4B) tried to mount a Longhorn PVC and got:

```
CSINode inchfield does not contain driver driver.longhorn.io
```

Fix: removed `systemManagedComponentsNodeSelector` entirely and cleared the persisted Longhorn setting:

```bash
kubectl patch settings.longhorn.io system-managed-components-node-selector \
  -n longhorn-system --type merge -p '{"value": ""}'
```

The setting persists in the CRD even after removing it from Helm values. You have to explicitly clear it.

### Mistake 2: Longhorn Manager on Control Plane Nodes

With the global node selector removed, the Longhorn Manager DaemonSet happily scheduled on all 16 nodes — including the 3 control plane nodes. Those promptly crashed:

```
level=fatal msg="Error starting manager: failed to check environment, please make
sure you have iscsiadm/open-iscsi installed on the host"
```

Talos control plane nodes don't have the `iscsi-tools` extension. Only workers do. And the control plane nodes had no taint to prevent the DaemonSet from scheduling there.

Wait. No taint? Aren't control plane nodes supposed to have `node-role.kubernetes.io/control-plane:NoSchedule`?

### Mistake 3: allowSchedulingOnControlPlanes Was True

Turns out `talconfig.yaml` had `allowSchedulingOnControlPlanes: true` — an explicit opt-in that tells Talos to **remove** the standard NoSchedule taint from control plane nodes. I'm not sure why I did this; perhaps I was concerned about nodes "only" being control plane, but really, the control plane nodes have enough to worry about with `etcd`.

So I set `allowSchedulingOnControlPlanes: false` in talconfig, regenerated configs, and applied.

```yaml
# talconfig.yaml
allowSchedulingOnControlPlanes: false
```

```bash
talhelper genconfig
talosctl apply-config --nodes 10.4.0.10 --file clusterconfig/goldentooth-allyrion.yaml --mode no-reboot
talosctl apply-config --nodes 10.4.0.11 --file clusterconfig/goldentooth-bettley.yaml --mode no-reboot
talosctl apply-config --nodes 10.4.0.12 --file clusterconfig/goldentooth-cargyll.yaml --mode no-reboot
```

Applied without reboot. Taints appeared immediately:

```
$ kubectl get nodes allyrion bettley cargyll -o custom-columns='NAME:.metadata.name,TAINTS:.spec.taints'
NAME       TAINTS
allyrion   [map[effect:NoSchedule key:node-role.kubernetes.io/control-plane]]
bettley    [map[effect:NoSchedule key:node-role.kubernetes.io/control-plane]]
cargyll    [map[effect:NoSchedule key:node-role.kubernetes.io/control-plane]]
```

With the taints in place, the `node.longhorn.io/worker` label workaround was no longer needed. The Longhorn manager's `nodeSelector` went back to empty — the taint does the exclusion automatically.

Existing pods on CP nodes (external-dns, metallb controller, kubevirt, etc.) aren't evicted — `NoSchedule` only prevents new scheduling. They'll migrate naturally when they restart. If we wanted force, it'd be `NoExecute`, but that's aggressive for a running cluster.

## The Final State

After the dust settled:

```
$ kubectl get ds -n longhorn-system
NAME                       DESIRED   READY   NODE SELECTOR
engine-image-ei-d91f5974   16        16      <none>
longhorn-csi-plugin        16        16      <none>
longhorn-manager           14        14      <none>
```

- **Longhorn Manager**: 14 pods — all worker nodes (CP excluded by taint)
- **CSI Plugin**: 16 pods — every node in the cluster (so any pod can mount volumes)
- **Engine Images**: 16 pods — every node

Storage:

```
$ kubectl get sc
NAME                 PROVISIONER             RECLAIMPOLICY   ALLOWVOLUMEEXPANSION
local-path           rancher.io/local-path   Delete          false
local-path-usb       rancher.io/local-path   Delete          true
longhorn (default)   driver.longhorn.io      Delete          true
```

Four Pi 5 NVMe nodes, each with ~931GB available, all schedulable in Longhorn. ~3.7TB raw, ~1.86TB usable with 2x replication. ServiceMonitor wired into prometheus-stack for metrics.

## Docker Registry: Back from the Dead

The docker registry had been in CrashLoopBackOff since SeaweedFS was torn down — it was configured to use S3 storage at `goldentooth-storage-filer.seaweedfs.svc.cluster.local:8333`, a service that no longer existed.

The fix was straightforward: switch from S3 to filesystem storage on a Longhorn PVC.

**Before** (S3/SeaweedFS):
```yaml
storage:
  s3:
    region: us-east-1
    regionendpoint: http://goldentooth-storage-filer.seaweedfs.svc.cluster.local:8333
    bucket: harbor-registry
```

**After** (filesystem/Longhorn):
```yaml
storage:
  filesystem:
    rootdirectory: /var/lib/registry
```

Added a 50Gi PVC, mounted it in the deployment, removed the S3 secret dependency. Flux reconciled, the PVC bound to a Longhorn volume, and the registry came back:

```
$ kubectl get pods -n docker-registry
NAME                               READY   STATUS    RESTARTS   AGE
docker-registry-74bd5d47b9-6hmzn   1/1     Running   0          12m
```

Sixteen hours of CrashLoopBackOff, resolved by swapping 10 lines of YAML. The registry now has replicated NVMe storage!
