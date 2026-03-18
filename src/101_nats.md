# NATS: Pub/Sub on the Bramble

## Why a Message Bus?

I've got storage. I've got observability. I've got security monitoring. I've got a service mesh worth of eBPF. What I don't have is a way for things on the cluster to *talk to each other* without going through HTTP like cavemen.

NATS is a message bus. Pub/sub, request-reply, fan-out — the whole deal. It's written in Go, which means it compiles to a single binary, starts in milliseconds, and uses almost no memory. That last part matters when your "servers" are Raspberry Pis with the thermal profile of a small toaster.

The plan: deploy it, poke at it, see how fast it goes on ARM hardware. No grand architecture, no event-driven microservice vision. Just "what does this thing do and how does it feel." JetStream (NATS' persistence layer) stays off for now — I want to understand the core pub/sub model before I start bolting on durability.

## The Deployment

### The Setup

Standard four-file Flux structure:

```
nats/
├── kustomization.yaml
├── namespace.yaml          # No PSA needed — runs unprivileged
├── repository.yaml         # HelmRepository → nats-io.github.io/k8s/helm/charts/
└── release.yaml            # HelmRelease with 3-node cluster config
```

Key values from the HelmRelease:

```yaml
config:
  cluster:
    enabled: true
    replicas: 3
  jetstream:
    enabled: false
  monitor:
    enabled: true
    port: 8222

container:
  image:
    repository: nats
    tag: "2.11-alpine"
  resources:
    requests:
      cpu: 50m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi

natsBox:
  enabled: true

promExporter:
  enabled: true
  port: 7777
  podMonitor:
    enabled: true
```

Three NATS nodes forming a full mesh cluster. Each pod runs three containers: the NATS server, a config reloader sidecar, and a Prometheus exporter. Plus a nats-box Deployment for CLI access.

Resource requests are tiny — 50m CPU and 128Mi per node. NATS genuinely doesn't need much. The entire 3-node cluster uses less RAM than a single Loki pod.

### The Helm Chart Values Schema Adventure

Here's a fun one. I wrote out a perfectly reasonable-looking `release.yaml`, had it reviewed, and then someone pointed out that the Helm chart values might not match what I wrote. "Might not" is doing a lot of heavy lifting here — Helm silently ignores unknown keys. So if you typo `container.resources` when the chart actually expects `container.image.resources`, your careful resource limits just... don't apply. No error. No warning. Your pods deploy with whatever defaults the chart author thought were reasonable.

`helm show values nats/nats` revealed:

1. The nats-box container has `resources` nested under `image` — as in `natsBox.container.image.resources`, not `natsBox.container.resources`. This is bizarre. Every other Helm chart I've seen puts resources as a sibling of image. But the NATS chart does its own thing, and if you don't check, you get nats-box pods with no resource limits running wild on your Pi 4Bs.

2. The nats-box image tag I'd picked (`0.14`) was ancient. The chart defaults to `0.19.2`. Dropped the override entirely.

3. The PodMonitor for Prometheus wasn't enabled by default. Added `promExporter.podMonitor.enabled: true` so the existing kube-prometheus-stack discovers and scrapes NATS metrics automatically.

Lesson learned (again): always `helm show values` before writing values files. Helm's silent-ignore behavior is a landmine.

### Deployment

Pushed to git, kicked Flux:

```
$ flux reconcile kustomization infrastructure --with-source
► annotating GitRepository flux-system in flux-system namespace
✔ fetched revision main@sha1:04beb2fd68bac6177277e0717c28950e75c08951
✔ applied revision main@sha1:04beb2fd68bac6177277e0717c28950e75c08951
```

Helm chart version `1.3.16` installed cleanly on the first try. No timeout issues, no CRD drama, no image pull failures. Genuinely the smoothest infrastructure deployment I've done on this cluster.

```
$ kubectl get pods -n nats
NAME                             READY   STATUS    RESTARTS   AGE
nats-nats-0                      3/3     Running   0          41s
nats-nats-1                      3/3     Running   0          41s
nats-nats-2                      3/3     Running   0          41s
nats-nats-box-778456b65f-4dpfj   1/1     Running   0          41s
```

Three NATS nodes, one nats-box. All running. 41 seconds. I don't trust it.

## Benchmarks

Time to see what Raspberry Pis can do with a real message bus.

### Single Publisher

```
$ nats bench pub test --msgs 1000000 --size 128
NATS Core NATS publisher stats: 151,413 msgs/sec ~ 18 MiB/sec ~ 6.60us
```

151K messages per second from a single publisher, 128-byte messages. Average latency 6.6 microseconds. On a Raspberry Pi. I've worked on production systems with worse numbers running on actual servers.

### Four Publishers

```
$ nats bench pub test --msgs 1000000 --size 128 --clients 4
NATS Core NATS publisher aggregated stats: 411,094 msgs/sec ~ 50 MiB/sec
  [1] 139,910 msgs/sec ~ 17 MiB/sec ~ 7.15us (250,000 msgs)
  [2] 122,697 msgs/sec ~ 15 MiB/sec ~ 8.15us (250,000 msgs)
  [3] 111,404 msgs/sec ~ 14 MiB/sec ~ 8.98us (250,000 msgs)
  [4] 102,794 msgs/sec ~ 12 MiB/sec ~ 9.73us (250,000 msgs)
  message rates min 102,794 | avg 119,201 | max 139,910 | stddev 13,884 msgs
  avg latencies min 7.15us | avg 8.50us | max 9.73us | stddev 0.96us
```

411K msgs/sec aggregate across 4 clients. Sub-10 microsecond latencies across the board. The throughput scales roughly linearly with publishers, which tracks — NATS has a zero-allocation hot path that basically just shuffles bytes between connections.

For context: these numbers are from *inside* the nats-box pod, talking to the NATS service over the cluster network (Cilium eBPF). Real cross-node pub/sub would add some network latency, but the message processing overhead is clearly not the bottleneck here.

## Observability

The Prometheus exporter is working — metrics are available on `:7777/metrics` from each NATS pod. The PodMonitor should get them into Prometheus automatically once the next scrape cycle hits. Standard NATS metrics: connection counts, message rates, byte throughput, slow consumers, route stats for the cluster mesh.

## JetStream: Adding Persistence

### The Config Change

Enabling JetStream is a one-section values change:

```yaml
config:
  jetstream:
    enabled: true
    fileStore:
      enabled: true
      dir: /data
      pvc:
        enabled: true
        size: 5Gi
        storageClassName: seaweedfs
```

5Gi per node, 15Gi total, backed by SeaweedFS. Should be plenty for experimenting.

### The StatefulSet Immutability Wall

Pushed the change. Flux tried to upgrade. Helm tried to patch the StatefulSet. Kubernetes said no.

```
error updating the resource "nats-nats":
  cannot patch "nats-nats" with kind StatefulSet: StatefulSet.apps "nats-nats"
  is invalid: spec: Forbidden: updates to statefulset spec for fields other than
  'replicas', 'ordinals', 'template', 'updateStrategy', 'revisionHistoryLimit',
  'persistentVolumeClaimRetentionPolicy' and 'minReadySeconds' are forbidden
```

Adding `volumeClaimTemplates` to an existing StatefulSet is an immutable field change. Kubernetes flat-out refuses it. Helm tried three times, rolled back three times, and the HelmRelease ended up in a rollback loop with a poisoned release history.

The fix was a full nuke-and-pave:

1. Delete the StatefulSet with `--cascade=orphan` (keeps pods running, but the rollback recreated it anyway)
2. Delete all Helm release secrets from `flux-system` namespace (`sh.helm.release.v1.nats-nats.v5` through `v9`)
3. Delete all orphaned resources in the `nats` namespace — pods, deployment, configmap, services, PDB, PodMonitor, secrets
4. Let Flux do a clean install from scratch

```
$ kubectl get pods -n nats
NAME                             READY   STATUS    RESTARTS   AGE
nats-nats-0                      3/3     Running   0          34s
nats-nats-1                      3/3     Running   0          34s
nats-nats-2                      3/3     Running   0          34s
nats-nats-box-778456b65f-ggsv9   1/1     Running   0          35s

$ kubectl get pvc -n nats
NAME                       STATUS   VOLUME         CAPACITY   STORAGECLASS   AGE
nats-nats-js-nats-nats-0   Bound    pvc-4baac...   5Gi        seaweedfs      34s
nats-nats-js-nats-nats-1   Bound    pvc-b37bb...   5Gi        seaweedfs      34s
nats-nats-js-nats-nats-2   Bound    pvc-d1f8e...   5Gi        seaweedfs      34s
```

Three PVCs, all bound to SeaweedFS. JetStream is live.

### Playing With Streams

Created a test stream with R3 replication across all three nodes:

```
$ nats stream add EVENTS --subjects "events.>" --retention limits \
    --max-age 1h --storage file --replicas 3

Stream EVENTS was created
  Subjects: events.>
  Replicas: 3
  Storage: File
  Cluster Group: S-R3F-RKzOy1A8
    Leader: nats-nats-2
    Replica: nats-nats-0, current
    Replica: nats-nats-1, current
```

Published 5 messages, created a pull consumer, replayed them all:

```
$ nats consumer next EVENTS replay --count 5
[01:50:48] subj: events.test / cons seq: 1 / str seq: 1 / pending: 4
message number 1 from the bramble
[01:50:48] subj: events.test / cons seq: 2 / str seq: 2 / pending: 3
message number 2 from the bramble
...
```

Messages go in, messages come back out. In order. With sequence numbers. Even after the publisher is long gone. This is the thing core pub/sub can't do.

### JetStream Benchmarks

Here's where it gets interesting. Same 4-client, 128-byte setup, but now with persistence:

| Mode | Throughput | Avg Latency |
|------|-----------|-------------|
| Core pub/sub | 411,094 msgs/sec | 8.5µs |
| JetStream async R1 | 4,144 msgs/sec | 951µs |
| JetStream async R3 | 1,007 msgs/sec | 3.9ms |
| JetStream sync R3 | 401 msgs/sec | 9.5ms |

That's a thousand-to-one ratio between core pub/sub and JetStream sync R3. Not a typo. Core NATS shuffles bytes in memory. JetStream sync R3 writes to disk on the leader, replicates to two followers via Raft consensus, waits for quorum acknowledgment from the SeaweedFS-backed PVCs, then responds. Every single message does a full consensus round-trip across the cluster network.

The async numbers are more forgiving — 1K msgs/sec at R3 is totally usable for event streams, audit logs, sensor data. You're batching the acks and letting the client pipeline ahead while the cluster catches up. R1 gets you 4K msgs/sec by skipping replication entirely, but then you've got a single point of failure, which kind of defeats the purpose.

For this cluster, R3 async is probably the sweet spot. Durable, replicated, and fast enough for anything I'd realistically run on a bramble.

## What's Next

Actually *building something* on top of NATS would be nice. A little event-driven app, maybe some sensor data pipeline across the bramble. Having a message bus with durable streams and nothing to stream is a very on-brand infrastructure hobby.
