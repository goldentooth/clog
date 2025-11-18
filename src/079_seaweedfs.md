# SeaweedFS: Distributed Object Storage (Take Two)

I needed distributed object storage for the cluster. Harbor needs an S3-compatible backend, and I'd rather not rely on external cloud storage for a local cluster. SeaweedFS is perfect for this – it's lightweight, designed for ARM, and provides S3-compatible APIs out of the box.

I've deployed SeaweedFS before ([chapter 63](./063_seaweedfs.md)), but that was ages ago, before the Talos migration. Time to do it properly with the operator pattern and USB SSD storage.

## The Architecture

SeaweedFS has three main components:

- **Masters** (3 replicas): Handle metadata and coordination using Raft consensus
- **Volume Servers** (6 replicas): Store the actual data, each on a dedicated USB SSD
- **Filer** (1 replica): Provides the S3-compatible API gateway

The plan: deploy everything via the SeaweedFS Operator, using `local-path-usb-provisioner` to provision storage on USB SSDs attached to 6 specific nodes.

## The Deployment

I reorganized the SeaweedFS directory structure into operator and cluster subdirectories, then deployed:

```yaml
# Master configuration (Raft HA)
master:
  replicas: 3
  config: |
    raftHashicorp = true
    defaultReplication = "001"  # 2 total copies

# Volume servers on USB SSDs
volume:
  replicas: 6
  nodeSelector:
    storage.seaweedfs/volume: "true"
  requests:
    storage: 100Gi
  storageClassName: local-path-usb

# Filer with S3 API
filer:
  replicas: 1
  s3:
    enabled: true
```

Flux picked it up, the operator deployed, masters came up, filer started... and then 4 of 6 volume servers were stuck Pending.

## The Problem: Inchfield's Corrupt Partition Table

Checking the pending volumes:

```bash
$ kubectl get pods -n seaweedfs | grep volume
goldentooth-storage-volume-0   0/1     Pending   0          2m
goldentooth-storage-volume-1   1/1     Running   0          2m
goldentooth-storage-volume-2   1/1     Running   0          2m
goldentooth-storage-volume-3   1/1     Running   0          2m
goldentooth-storage-volume-4   0/1     Pending   0          2m
goldentooth-storage-volume-5   1/1     Running   0          2m
```

Both pending volumes were scheduled to `inchfield`. The PVCs were bound, but the helper pods that format the volumes were failing:

```bash
$ kubectl logs helper-pod-create-pvc-... -n local-path-usb-provisioner
mkdir: can't create directory '/var/mnt/usb/...': Read-only file system
```

Read-only filesystem? That's weird. The Talos volume manager should have mounted `/var/mnt/usb` from the USB disk. Let me check:

```bash
$ talosctl -n inchfield get volumestatuses
NAMESPACE   TYPE           ID      VERSION   PHASE   LOCATION
runtime     VolumeStatus   u-usb   3         failed  /dev/sda1

$ talosctl -n inchfield get volumestatuses u-usb -o yaml
spec:
  phase: failed
  error: "error probing disk: open /dev/sda1: no such file or directory"
```

The volume manager sees the partition in its discovery scan, but `/dev/sda1` doesn't exist as a device node. That's a partition table problem.

Looking at the kernel's view:

```bash
$ talosctl -n inchfield read /proc/partitions | grep sda
   8        0  976762584 sda
```

No `sda1` partition! Compare with a working node:

```bash
$ talosctl -n gardener read /proc/partitions | grep sda
   8        0  117220824 sda
   8        1  117219328 sda1
```

The kernel can't see any partitions on inchfield's disk. The GPT partition table is corrupt or missing.

## The Fix: Nuke It From Orbit

Time to rebuild the disk. I created a privileged pod on inchfield to partition and format the disk:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: disk-formatter-inchfield
  namespace: local-path-usb-provisioner
spec:
  nodeSelector:
    kubernetes.io/hostname: inchfield
  restartPolicy: Never
  hostNetwork: true
  hostPID: true
  securityContext:
    privileged: true
  containers:
  - name: formatter
    image: ubuntu:22.04
    command: ["/bin/bash", "-c"]
    args: ["apt-get update && apt-get install -y gdisk xfsprogs && sleep 3600"]
    volumeMounts:
    - name: dev
      mountPath: /dev
  volumes:
  - name: dev
    hostPath:
      path: /dev
```

But when I tried to format, Talos Volume Manager had the device locked. Even after wiping the partition table, the kernel kept using the GPT backup table at the end of the disk.

### Attempt 1: Wipe just the beginning
```bash
$ kubectl exec disk-formatter-inchfield -- dd if=/dev/zero of=/dev/sda bs=1M count=100
```

Nope. Partition still there (restored from backup GPT).

### Attempt 2: Properly zap both GPT tables
```bash
$ kubectl exec disk-formatter-inchfield -- sgdisk --zap-all /dev/sda
GPT data structures destroyed!
Warning: The kernel is still using the old partition table.
```

Closer, but the kernel and Talos still had locks on the device.

### Attempt 3: Reboot the node

```bash
$ talosctl -n inchfield reboot
```

After the reboot, I checked the disk:

```bash
$ talosctl -n inchfield get discoveredvolumes | grep usb
inchfield   runtime     DiscoveredVolume   sda1   2   partition   1.0 TB   xfs   u-usb
```

Wait, what? It's already XFS with the `u-usb` label?

Turns out there was an old XFS filesystem on the disk from a previous setup. The corrupt GPT was just hiding it. The reboot cleared Talos' locks and allowed it to discover the filesystem properly.

## The Second Problem: Permission Denied

With the disk working, the volume pods started... and immediately crashed:

```bash
$ kubectl logs goldentooth-storage-volume-0 -n seaweedfs
Folder /data0 Permission: -rwxr-xr-x
F1118 03:27:47 cannot generate uuid of dir /data0: failed to write uuid
  to /data0/vol_dir.uuid: open /data0/vol_dir.uuid: permission denied
```

The helper pod created the directory as root with 755 permissions, but SeaweedFS runs as uid 1000 (non-root). Checking a working volume:

```bash
$ kubectl exec goldentooth-storage-volume-1 -n seaweedfs -- ls -ld /data0
drwxrwxrwx    2 root     root           143 Nov 18 03:05 /data0
```

777 permissions on working volumes, 755 on inchfield's. The helper pod on inchfield must have created it with restrictive permissions.

Quick fix with another privileged pod:

```bash
$ kubectl run permission-fixer --rm -i --restart=Never \
  --overrides='{"spec":{"nodeSelector":{"kubernetes.io/hostname":"inchfield"},"hostNetwork":true,"containers":[{"name":"fixer","image":"busybox","command":["sh","-c","chmod -R 777 /var/mnt/usb"],"securityContext":{"privileged":true},"volumeMounts":[{"name":"usb","mountPath":"/var/mnt/usb"}]}],"volumes":[{"name":"usb","hostPath":{"path":"/var/mnt/usb"}}]}}' \
  --image=busybox -n local-path-usb-provisioner
Permissions fixed!

$ kubectl delete pods goldentooth-storage-volume-0 goldentooth-storage-volume-4 -n seaweedfs
```

## Success

A minute later:

```bash
$ kubectl get pods -n seaweedfs
NAME                           READY   STATUS    RESTARTS       AGE
goldentooth-storage-filer-0    1/1     Running   0              36m
goldentooth-storage-master-0   1/1     Running   1 (36m ago)    36m
goldentooth-storage-master-1   1/1     Running   1 (36m ago)    36m
goldentooth-storage-master-2   1/1     Running   1 (36m ago)    36m
goldentooth-storage-volume-0   1/1     Running   1              2m
goldentooth-storage-volume-1   1/1     Running   1              29m
goldentooth-storage-volume-2   1/1     Running   1              29m
goldentooth-storage-volume-3   1/1     Running   1              29m
goldentooth-storage-volume-4   1/1     Running   1              2m
goldentooth-storage-volume-5   1/1     Running   1              29m
```

All green! The cluster now has:

- **600GB** of distributed object storage across 6 nodes
- **S3-compatible API** ready for Harbor
- **Automatic replication** (2 copies of each object)
- **Fault tolerance** via Raft consensus

## Key Learnings

1. **GPT has backup tables** – Wiping just the beginning of a disk isn't enough. GPT keeps a backup partition table at the end, and the kernel will restore from it. Use `sgdisk --zap-all`.

2. **Talos Volume Manager is persistent** – Even after wiping partition data, Talos caches volume information. A reboot was needed to fully release locks.

3. **local-path provisioner permission issues** – Helper pods run as root and create directories with restrictive permissions. Applications running as non-root need 777 permissions on the mount point.

4. **Partition table corruption is sneaky** – Talos' DiscoveredVolumes controller scans for filesystem UUIDs directly, so it can "see" filesystems even when the partition table is corrupt. But without valid partition entries, the kernel won't create device nodes, preventing actual mounts.
