# Loki and Alloy: Log Aggregation for the Bramble

I've had Prometheus and Grafana running for a while now, happily scraping metrics from everything. But metrics only tell part of the story. When something breaks, I want to see the actual log output – not just a spike on a graph.

Time to add centralized logging. The obvious choice in the Grafana ecosystem: **Loki**.

## Why Loki Instead of ELK?

Elasticsearch is powerful but *expensive*. It indexes every word in every log line, which means:
- Massive CPU to build indexes
- Massive storage (indexes can be 2-3x the raw log size)
- Massive memory to keep queries fast

For a Raspberry Pi cluster? That would melt my little bramble.

Loki takes a radically different approach: **index nothing**. Well, almost nothing. Loki only indexes *labels* (key-value pairs like `namespace=monitoring`, `pod=grafana`), not log content. The actual log text just gets compressed and stored.

When you query, Loki:
1. Uses labels to narrow down which "chunks" of logs to look at (fast label lookups)
2. Decompresses and greps through those chunks

The tradeoff: searching for specific text across huge time ranges is slower. But filtering by labels is instant, and for structured logging that's usually what you want anyway.

The other big advantage: Loki uses the same label model as Prometheus. The labels you use to identify services in Prometheus (`app=grafana`, `namespace=monitoring`) work identically in Loki. Same queries, same mental model, same Grafana dashboards can show metrics *and* logs side by side.

## Architecture Decisions

Loki can run in several modes:
- **Monolithic** – everything in one binary
- **Simple Scalable** – read and write paths separated
- **Microservices** – every component separate

For a homelab? Monolithic. I don't need to be a Loki SME, I just need logs.

For storage, I initially considered using SeaweedFS S3 (already running), but decided on local filesystem instead. If the Loki pod dies, I lose recent logs – but for debugging current issues, I don't need months of historical data. Simple wins.

## The Log Shipper: Alloy

Loki doesn't collect logs itself – it just receives them. You need a shipper. The old-school choice was **Promtail**, but Grafana replaced it with **Alloy** – a unified observability agent that can collect logs, metrics, and traces.

Alloy uses a pipeline configuration language called River:

```river
discovery.kubernetes "pods" { role = "pod" }
loki.source.kubernetes "pods" {
  targets    = discovery.kubernetes.pods.targets
  forward_to = [loki.write.default.receiver]
}
loki.write "default" {
  endpoint { url = "http://loki:3100/loki/api/v1/push" }
}
```

Components connect together like building blocks. Discover pods → collect their logs → ship to Loki.

## The Deployment

Created the GitOps structure:

```
gitops/infrastructure/
├── loki/
│   ├── kustomization.yaml
│   ├── repository.yaml      # HelmRepository for Grafana charts
│   ├── release.yaml         # Loki HelmRelease
│   └── datasource.yaml      # Grafana datasource ConfigMap
└── alloy/
    ├── kustomization.yaml
    └── release.yaml         # Alloy HelmRelease
```

The Grafana datasource uses a neat trick: Grafana's sidecar watches for ConfigMaps with the label `grafana_datasource: "1"` and auto-provisions them. No need to modify the prometheus-stack values.

## The Problems: A Debugging Odyssey

### Problem 1: Chart Values Structure Changed in 6.x

Deployed Loki. Got way more pods than expected:

```
monitoring-loki-0                 2/2     Running
loki-canary-xxxxx                 1/1     Running   (x17!)
monitoring-loki-chunks-cache-0    2/2     Running
monitoring-loki-results-cache-0   2/2     Running
```

I had disabled the canary and caches in my values:

```yaml
monitoring:
  lokiCanary:
    enabled: false
```

But the Loki chart 6.x moved `lokiCanary` to the **root level**, and enabled memcached caches **by default**. My config was under the deprecated `monitoring` section, so it was ignored.

The fix:

```yaml
# Root level, not under monitoring!
lokiCanary:
  enabled: false
chunksCache:
  enabled: false
resultsCache:
  enabled: false
```

### Problem 2: The File Path Construction Nightmare

My initial Alloy config used `loki.source.file` to read log files from disk. This requires building the correct path to each container's logs:

```
/var/log/pods/{namespace}_{pod-name}_{pod-uid}/{container}/*.log
```

Four pieces of information, concatenated with underscores and slashes. I tried using Prometheus-style relabeling:

```river
rule {
  source_labels = ["__meta_kubernetes_namespace", "__meta_kubernetes_pod_name",
                   "__meta_kubernetes_pod_uid", "__meta_kubernetes_pod_container_name"]
  separator     = "/"
  regex         = "(.*)/(.*)/(.*)/(.*)"
  replacement   = "/var/log/pods/$1_$2_$3/$4/*.log"
}
```

First attempt: paths came out as `/var/log/pods/*abc123/container/*.log`. The asterisk was being interpreted literally.

Second attempt: paths missing namespace and pod name. Greedy regex `(.*)` was eating too much.

Third attempt: used `[^/]*` instead of `.*` to match "anything except slashes":

```river
regex = "([^/]*)/([^/]*)/([^/]*)/([^/]*)"
```

Paths finally looked correct! But still no logs flowing.

### Problem 3: PodSecurity Strikes Again

```
Error creating: pods "monitoring-alloy-xxx" is forbidden:
violates PodSecurity "baseline:latest": hostPath volumes (volume "varlog")
```

The `monitoring` namespace has PodSecurity set to `baseline`, which blocks hostPath volume mounts. Alloy needed to mount `/var/log` from the host to read log files.

Options:
1. Label the namespace as `privileged` (YOLO)
2. Use the Kubernetes API instead of file access

Option 1 would work but feels wrong – the whole monitoring namespace would lose security restrictions just for log collection.

### The Solution: Kubernetes API Method

Turns out Alloy has `loki.source.kubernetes` which streams logs directly from the Kubernetes API instead of reading files. No hostPath needed, works with any PodSecurity policy.

```river
discovery.kubernetes "pods" {
  role = "pod"
}

discovery.relabel "pods" {
  targets = discovery.kubernetes.pods.targets
  // ... label extraction rules ...
}

loki.source.kubernetes "pods" {
  targets    = discovery.relabel.pods.output
  forward_to = [loki.process.pods.receiver]
}
```

Bonus: no need for a DaemonSet! Since we're reading from the API (not local files), a Deployment with 2 replicas + clustering works fine. Fewer pods, simpler setup.

### Problem 4: Service Name Mismatch

Logs still not flowing. Checked Alloy logs:

```
error="Post \"http://loki.monitoring.svc.cluster.local:3100/...\":
       lookup loki.monitoring.svc.cluster.local: no such host"
```

Checked the actual service:

```bash
$ kubectl -n monitoring get svc | grep loki
monitoring-loki    ClusterIP   10.106.228.167   <none>   3100/TCP
```

The Helm chart names the service `{release-name}-{chart-name}`. My release was `loki` in namespace `monitoring`, so Helm created `monitoring-loki`. Not `loki`.

Fixed the Alloy config and Grafana datasource to use `monitoring-loki.monitoring.svc.cluster.local`.

## The Final Setup

```
┌─────────────────┐     ┌──────────────┐     ┌─────────────┐
│  Kubernetes     │     │    Alloy     │     │    Loki     │
│    API          │────▶│  (2 replicas │────▶│ (monolithic)│
│ (pod logs)      │     │  clustered)  │     │             │
└─────────────────┘     └──────────────┘     └──────┬──────┘
                                                     │
                                                     ▼
                                              ┌─────────────┐
                                              │   Grafana   │
                                              │  (Explore)  │
                                              └─────────────┘
```

Verified logs are flowing:

```bash
$ curl -s "http://localhost:3100/loki/api/v1/labels"
{"status":"success","data":["app","container","instance","job","namespace","node","pod","service_name"]}
```

In Grafana Explore, `{job="kubernetes-pods"}` returns logs from across the cluster.
