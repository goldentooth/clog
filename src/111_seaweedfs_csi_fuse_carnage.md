# SeaweedFS CSI: The FUSE Mount Massacre

## The Setup

Routine alert triage. SeaweedFS CSI DaemonSet was flagged as "rollout stuck" — the mount pods were running mixed image tags. Some on `v1.4.5`, some on `latest` (which resolved to an older build). The CSI DaemonSet uses an `OnDelete` update strategy, which means pods don't auto-update when the spec changes. You have to manually delete each pod to pick up the new image. This is by design — you don't want FUSE mount daemons restarting under active volumes.

So I did a careful rolling restart. Phase one: the seven nodes with no active PVC consumers. Phase two: the four nodes hosting pods with SeaweedFS-backed PVCs (Prometheus, Alertmanager, Loki, Tempo, and some others). All 13 mount pods came up on `v1.4.5`. The DaemonSet went green. Alert cleared.

And then everything caught fire.

## Transport Endpoint Is Not Connected

Within minutes, Alertmanager started screaming: `KubeletDown`, `KubeAPIDown`. Both critical. Both saying "Target disappeared from Prometheus target discovery."

First thought: the cluster is down. Checked node status — all 17 nodes Ready. Checked the API — responding fine (how else would I be running kubectl?). Checked Prometheus workloads — all pods Running with correct replica counts.

Then I queried Prometheus directly:

```
up{job="kubelet"} → empty
up{job="apiserver"} → empty
count by (job) (up) → empty
prometheus_build_info → empty
```

*Every* query returned empty. Prometheus was running but had zero data. Pulled the logs:

```
ts=2026-03-25T21:45:23.621Z level=error component="scrape manager"
  msg="Scrape commit failed"
  err="write to WAL: log samples: write /prometheus/wal/00000393:
  transport endpoint is not connected"
```

`ENOTCONN` — the Linux kernel's way of saying "this FUSE mount is dead and the daemon that was serving it is gone." Every single scrape across every target was failing to write to the WAL. Prometheus was collecting metrics over the network just fine, but couldn't persist them to disk.

Checked Alertmanager, Loki, and Tempo — all the same:

```
# Alertmanager
chdir to cwd ("/alertmanager"): transport endpoint is not connected

# Loki
write /var/loki/wal/00003094: transport endpoint is not connected

# Tempo
open /var/tempo/traces: transport endpoint is not connected
```

Every SeaweedFS-backed statefulset in the monitoring namespace had a dead FUSE mount. Four out of four.

## What Happened

When a SeaweedFS CSI mount pod restarts, the kernel-side FUSE mount loses its userspace connection. The mount point still exists in the filesystem namespace — it's still in the kernel's mount table — but any I/O to it immediately returns `ENOTCONN`. The new CSI mount pod starts fresh and establishes new FUSE mounts for new volume requests, but it doesn't re-attach to existing mounts from the old pod. It can't — those mounts are kernel state tied to the old process.

This is the exact scenario the `OnDelete` update strategy was designed to prevent. The idea is: you delete the mount pod, then immediately restart the consumer pods so they get fresh mounts from the new daemon. But I did the rolling restart in the wrong order — I updated the mount pods and then... didn't restart the consumers. The monitoring pods sat there with their dead FUSE mounts, unable to write, firing alerts about targets disappearing from discovery.

## The Recovery

### Phase 1: The Easy Ones

Deleted all four monitoring statefulset pods:

```bash
kubectl delete pod -n monitoring \
  prometheus-monitoring-kube-prometheus-prometheus-0 \
  alertmanager-monitoring-kube-prometheus-alertmanager-0 \
  monitoring-loki-0 \
  monitoring-tempo-0
```

The StatefulSet controller recreated them. Alertmanager landed on fenn, Loki on inchfield, Tempo on harlton. All three got rescheduled to different nodes than before, which meant new VolumeAttachments, new FUSE mounts, clean start. They came up fine.

Prometheus got rescheduled back to payne. Same node, same stale mount path. The kubelet tried to mount the volume and hit:

```
MountVolume.MountDevice failed: mkdir .../globalmount: file exists
```

The stale FUSE mount point was still sitting in the kubelet's CSI plugin directory. The directory existed, but it pointed at a dead FUSE daemon. The kubelet couldn't create it (it already exists) and couldn't use it (it's dead).

### Phase 2: The Multi-Attach Problem

The three pods that moved to new nodes hit a different issue:

```
Multi-Attach error for volume "pvc-8b86c8c4-..."
Volume is already exclusively attached to one node and can't be attached to another
```

The old VolumeAttachments still pointed at the original nodes. RWO volumes can only attach to one node at a time, and the stale attachments were blocking the new ones. Fixed by deleting the four VolumeAttachments — safe because the old mounts were already dead.

Alertmanager, Loki, and Tempo came up after that. Prometheus remained stuck on payne.

### Phase 3: The Prometheus Odyssey

The `globalmount: file exists` error meant the stale FUSE mount point needed manual cleanup. Reached into the CSI mount pod on payne (which has `/var/lib/kubelet/plugins` host-mounted) and unmounted the dead FUSE:

```bash
kubectl exec -n seaweedfs seaweedfs-seaweedfs-csi-driver-mount-zcmzj -- \
  umount /var/lib/kubelet/plugins/kubernetes.io/csi/seaweedfs-csi-driver/\
  0cb68b644c2fe52cc3e0a05245c524eba0a80fd8c2b144ce5fe6b4680fa64822/globalmount

kubectl exec -n seaweedfs seaweedfs-seaweedfs-csi-driver-mount-zcmzj -- \
  rmdir /var/lib/kubelet/plugins/kubernetes.io/csi/seaweedfs-csi-driver/\
  0cb68b644c2fe52cc3e0a05245c524eba0a80fd8c2b144ce5fe6b4680fa64822/globalmount
```

Prometheus pod got recreated, volume attached, mount attempted... and hit:

```
open /prometheus/queries.active: permission denied
panic: Unable to create mmap-ed active query log
```

The fresh FUSE mount was owned by root:root. Prometheus runs as uid 1000, gid 2000. Fixed the ownership via the CSI mount pod:

```bash
kubectl exec -n seaweedfs ... -- chown 1000:2000 .../globalmount
kubectl exec -n seaweedfs ... -- chmod 775 .../globalmount
```

Still crashed. FUSE mounts don't reliably honor `chown` — the FUSE daemon controls access, and `default_permissions` mode uses the kernel's permission checking against the UID the daemon reports, not what you set with `chown`. The `chown` appeared to work (the directory showed `1000:2000` in `ls -la`) but the FUSE daemon still rejected writes.

Deleted the pod again, deleted the VolumeAttachment again, and nuked the entire stale CSI volume directory:

```bash
kubectl exec -n seaweedfs ... -- rm -rf \
  /var/lib/kubelet/plugins/kubernetes.io/csi/seaweedfs-csi-driver/\
  0cb68b644c2fe52cc3e0a05245c524eba0a80fd8c2b144ce5fe6b4680fa64822
```

Now the CSI driver should recreate the whole thing from scratch. Except it didn't. The CSI node plugin went straight to `NodePublishVolume` (bind mount) without running `NodeStageVolume` (FUSE mount) first. The kubelet had cached the "this volume is already staged" state in memory, so it skipped the staging step entirely. The bind mount failed because the source directory didn't exist.

The fix: restart the CSI **node** pod on payne (not the mount pod — the node pod handles the gRPC lifecycle):

```bash
kubectl delete pod -n seaweedfs seaweedfs-seaweedfs-csi-driver-node-c9zpg
```

This forced the kubelet to re-register the CSI driver and re-run the full attach → stage → publish flow. The new CSI node pod came up, NodeStageVolume ran, created the globalmount directory, established a fresh FUSE mount, NodePublishVolume bind-mounted it into the pod, and Prometheus finally started.

## The Aftermath

All four monitoring pods running. Prometheus scraping all targets. Alertmanager back to just the Watchdog canary. Loki ingesting logs. Tempo polling traces.

Prometheus lost its historical TSDB data — the fresh FUSE mount came up empty. The data still exists in SeaweedFS (the volume bucket wasn't deleted), but the WAL state was corrupt from all the `ENOTCONN` writes, and the new mount apparently didn't recover the old blocks. This is a known limitation of FUSE-backed Prometheus — the WAL isn't crash-safe when the underlying mount disappears mid-write. The TSDB will rebuild from new scrapes. A week of historical data, gone. Not ideal, but not catastrophic.

## The CSI FUSE Recovery Playbook

For future me, the full recovery chain when a SeaweedFS FUSE mount goes stale:

1. **Unmount the dead FUSE**: `umount` via a pod with host access
2. **Remove the stale directory**: `rm -rf` the entire CSI volume hash directory
3. **Delete the VolumeAttachment**: So the CSI controller re-attaches
4. **Restart the CSI node pod on that host**: To clear the kubelet's "already staged" cache
5. **Delete the consumer pod**: So it gets recreated with fresh mounts

Steps 1-4 must all happen. Skipping any one of them leaves you stuck in a different failure mode. I learned this the hard way, one step at a time.

## Lessons

The `OnDelete` DaemonSet strategy is a trap that teaches you a lesson every time you interact with it. It exists for a good reason — you don't want FUSE mounts disappearing under active workloads — but it creates a coordination problem: you need to restart consumers immediately after restarting the mount daemon. There's no grace period. The moment the old mount pod dies, every volume it was serving becomes an `ENOTCONN` time bomb.

The correct procedure for a SeaweedFS CSI rolling restart: for each node, delete the mount pod, wait for the new one, then immediately bounce every pod on that node that uses a SeaweedFS volume. Not "later." Not "after all mount pods are updated." *Immediately*, per-node, in lockstep. I did the mount pods first and the consumers never, which is the one ordering that guarantees maximum carnage.

Also: the kubelet's CSI staging cache is invisible and persistent. If you clean up a CSI volume's globalmount directory, the kubelet still thinks the volume is staged and will skip NodeStageVolume on the next mount attempt. The only way to clear that cache is to restart the CSI node pod, which forces driver re-registration. This is not documented anywhere I could find.
