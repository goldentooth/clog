# NATS: Pub/Sub on the Bramble

## Why a Message Bus?

I've got storage. I've got observability. I've got security monitoring. I've got a service mesh worth of eBPF. What I don't have is a way for things on the cluster to *talk to each other* without going through HTTP like cavemen.

NATS is a message bus. Pub/sub, request-reply, fan-out — the whole deal. It's written in Go, which means it compiles to a single binary, starts in milliseconds, and uses almost no memory. That last part matters when your "servers" are Raspberry Pis with the thermal profile of a small toaster.

The plan: deploy it, poke at it, see how fast it goes on ARM hardware. No grand architecture, no event-driven microservice vision. Just "what does this thing do and how does it feel." JetStream (NATS' persistence layer) stays off for now — I want to understand the core pub/sub model before I start bolting on durability.

## The Deployment

### Wait, Don't I Already Have Logging?

Funny story. I originally had "Fluentd" on the TODO list. Then I looked at what was already running and realized I had Alloy → Loki → Grafana doing the exact same thing Fluentd would do. The TODO item was from before I set up the Alloy pipeline back in entry #086. So that one just got a line through it.

NATS, on the other hand, fills a gap nothing else covers. There's no messaging layer on the bramble. Everything communicates via HTTP APIs or not at all.

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

## What's Next

JetStream is the obvious next step — persistent streams with replay, consumer groups, exactly-once delivery. That'll need SeaweedFS-backed PVCs, which adds complexity but also makes NATS useful for things that can't afford to lose messages.

Beyond that, actually *building something* on top of NATS would be nice. A little event-driven app, maybe some sensor data pipeline across the bramble. Having a message bus is cool. Having a message bus with nothing to say is... an infrastructure hobby, which I suppose is what this entire project is.
