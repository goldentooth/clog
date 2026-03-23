# Flux Status and ntfy Notifications: The MCP Server Learns to Read GitOps

## Why

The MCP server could already tell me about nodes, pods, workloads, certs, alerts, logs, and metrics. But it couldn't answer "is Flux actually reconciling everything?" or "did ntfy fire any notifications recently?" — which are arguably the two most important questions when you're about to pile new changes onto the cluster.

Flux is the entire deployment mechanism. If a Kustomization is stuck or a HelmRelease is failing, nothing I push is going to land. And ntfy is where all the cluster alerts end up. Not being able to query either of these through the MCP server felt like a gap worth closing.

## Flux Status Tools

### The CRD Zoo

Flux has a *lot* of CRDs spread across four API groups:

| API Group                         | Resources                                           |
| --------------------------------- | --------------------------------------------------- |
| `kustomize.toolkit.fluxcd.io/v1`  | Kustomization                                       |
| `helm.toolkit.fluxcd.io/v2`       | HelmRelease                                         |
| `source.toolkit.fluxcd.io/v1`     | GitRepository, HelmRepository, OCIRepository        |
| `image.toolkit.fluxcd.io/v1beta2` | ImageRepository, ImagePolicy, ImageUpdateAutomation |

None of these are in `k8s-openapi`, so they all go through kube-rs's dynamic API — same approach as the cert-manager tool from the previous round.

I split the Flux tools into three, because dumping all of this into one response would be overwhelming:

1. **`get_flux_status`** — The reconciliation tools: Kustomizations and HelmReleases. These are the "is my stuff deploying?" tools. Returns ready state, last applied revision, source reference, and path.

2. **`get_flux_sources`** — Where Flux pulls from: GitRepository, HelmRepository, OCIRepository. Returns sync state, URL, and latest artifact revision.

3. **`get_flux_images`** — The image automation pipeline: ImageRepository (what tags exist?), ImagePolicy (which tag should we use?), ImageUpdateAutomation (did it push a commit?). Returns scan times, tag counts, latest image selections, and last push commits.

### Shared Status Extraction

Every Flux resource follows the same status convention — a `conditions` array with a `Ready` condition that has `status`, `message`, and `lastTransitionTime`. I pulled this into a shared helper:

```rust
fn flux_status_summary(data: &serde_json::Value) -> serde_json::Value {
    let ready = data
        .pointer("/status/conditions")
        .and_then(|c| c.as_array())
        .and_then(|conditions| {
            conditions.iter().find(|c|
                c.get("type").and_then(|t| t.as_str()) == Some("Ready"))
        });
    // ... extract status, message, lastTransitionTime
}
```

The `get_flux_status` tool also calculates a `not_ready_count` across both Kustomizations and HelmReleases, so I can quickly see if anything's broken without reading through every resource.

### Concurrent API Calls

Each tool queries multiple CRD types, so I used `tokio::join!` to fire them in parallel:

```rust
let lp = ListParams::default();
let (ks_result, hr_result) = tokio::join!(
    ks_api.list(&lp),
    hr_api.list(&lp),
);
```

One fun detail: you can't write `ks_api.list(&ListParams::default())` inside `tokio::join!` — the temporary `ListParams` gets dropped before the future resolves. The borrow checker catches it, thankfully. Binding to `let lp` extends the lifetime.

## ntfy Notifications

### The Poll API

ntfy has a clever HTTP API for retrieving historical messages. Instead of the streaming SSE endpoint, you add `?poll=1` to get all matching messages as newline-delimited JSON and close the connection:

```
GET http://ntfy.ntfy.svc:80/cluster-alerts/json?poll=1&since=24h
```

Each line is a separate JSON object. Most are `"event": "message"` but there are also keepalive events, so the tool filters to only actual messages.

The `get_notifications` tool accepts a topic name and an optional `since` parameter (defaults to `24h`). Returns title, message body, priority, tags, and Unix timestamp for each notification.

### RBAC

The Flux tools needed new RBAC permissions since we're querying CRDs that weren't in the original ClusterRole. Added read access to all four Flux API groups:

```yaml
- apiGroups: ["kustomize.toolkit.fluxcd.io"]
  resources: [kustomizations]
  verbs: ["get", "list", "watch"]
- apiGroups: ["helm.toolkit.fluxcd.io"]
  resources: [helmreleases]
  verbs: ["get", "list", "watch"]
- apiGroups: ["source.toolkit.fluxcd.io"]
  resources: [gitrepositories, helmrepositories, ocirepositories]
  verbs: ["get", "list", "watch"]
- apiGroups: ["image.toolkit.fluxcd.io"]
  resources: [imagerepositories, imagepolicies, imageupdateautomations]
  verbs: ["get", "list", "watch"]
```

Pushed RBAC to gitops before the MCP code — same ordering trick as last time. Flux reconciles the RBAC in ~5 minutes, the MCP build takes ~9 minutes through the CI pipeline, so the permissions are guaranteed to exist before the new pod starts trying to use them.

## The Full Tool Set

The server now exposes 14 tools:

| Tool                | Source       | What it does                                    |
| ------------------- | ------------ | ----------------------------------------------- |
| `get_version`       | Static       | Server version and build SHA                    |
| `get_node_status`   | K8s API      | Node readiness, CPU, memory, OS                 |
| `get_pods`          | K8s API      | Pod phase, restarts, node placement             |
| `get_namespaces`    | K8s API      | Namespace listing                               |
| `get_events`        | K8s API      | Recent cluster events                           |
| `get_workloads`     | K8s API      | Deployment/StatefulSet/DaemonSet replicas       |
| `get_certificates`  | K8s API      | cert-manager certificate status and expiry      |
| `get_alerts`        | Alertmanager | Active alerts with severity                     |
| `query_logs`        | Loki         | LogQL log search                                |
| `query_metrics`     | Prometheus   | PromQL metric queries                           |
| `get_flux_status`   | K8s API      | Kustomization and HelmRelease reconciliation    |
| `get_flux_sources`  | K8s API      | GitRepository/HelmRepository/OCIRepository sync |
| `get_flux_images`   | K8s API      | Image automation pipeline status                |
| `get_notifications` | ntfy         | Recent notifications from any topic             |

That's a pretty complete read-only view of the cluster. I can ask about infrastructure state, deployment pipeline status, observability data, and notification history — all through natural language via Claude Code. The only thing missing at this point is write operations, but for those we can just shell out to `kubectl`.
