# MCP Server: From Static JSON to Cluster Eyes

## The Starting Point

The MCP server was running on the cluster, exposed at `mcp.goldentooth.net`, and Claude Code could talk to it. But all it could do was return its own version number and a hardcoded list of node names. Useful for proving the pipeline worked, not useful for actually managing the bramble.

The goal: give the MCP server real access to the cluster so it could answer questions about what's actually happening — node health, pod status, workload state, cert expiry, active alerts, logs, metrics. The full observability stack, queryable through natural language via Claude Code.

## In-Cluster Kubernetes API Access

### RBAC

Kubernetes doesn't let pods talk to the API server by default — you need a ServiceAccount with explicit permissions. Created a read-only ClusterRole:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: goldentooth-mcp-reader
rules:
  - apiGroups: [""]
    resources: [nodes, pods, services, namespaces, events, configmaps]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: [deployments, statefulsets, daemonsets]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["batch"]
    resources: [jobs, cronjobs]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["cert-manager.io"]
    resources: [certificates, clusterissuers, issuers, certificaterequests]
    verbs: ["get", "list", "watch"]
```

Read-only across the entire cluster. The MCP server can see everything but touch nothing. Bound to a ServiceAccount in the `goldentooth-mcp` namespace via a ClusterRoleBinding.

One important ordering detail: I pushed the RBAC changes to the gitops repo *before* pushing the MCP code that depends on it. Flux reconciles gitops in ~5 minutes, and the MCP build takes ~9 minutes through the full CI pipeline. So the ServiceAccount and token are guaranteed to exist by the time the new pod starts. If you get this backwards, `kube::Client::try_default()` fails because there's no service account token mounted yet.

### kube-rs

Added `kube` and `k8s-openapi` to the Rust dependencies. The `kube` crate's `Client::try_default()` automatically detects the in-cluster environment — it reads the service account token from `/var/run/secrets/kubernetes.io/serviceaccount/token` and the API server CA cert from the same directory. No configuration needed.

```rust
let kube_client = match Client::try_default().await {
    Ok(client) => {
        tracing::info!("Kubernetes client initialized (in-cluster)");
        Some(client)
    }
    Err(e) => {
        tracing::warn!("Kubernetes client not available: {e}");
        None
    }
};
```

The client is `Option<Client>` so the server degrades gracefully when running locally for development — the cluster tools return a clear error instead of panicking.

### rmcp Parameter Handling

Discovered that `rmcp`'s `#[tool]` macro uses a `Parameters<T>` wrapper for tool inputs, not a `#[tool(param)]` attribute:

```rust
#[tool(description = "List pods. Optionally filter by namespace.")]
async fn get_pods(
    &self,
    Parameters(input): Parameters<NamespaceFilter>,
) -> Result<CallToolResult, McpError> {
    cluster::get_pods(self.require_kube()?, input.namespace.as_deref()).await
}
```

The `NamespaceFilter` struct derives both `Deserialize` and `schemars::JsonSchema`, which lets the MCP protocol auto-generate the parameter schema that Claude Code uses for tool discovery.

### k8s-openapi Version Dance

Ran into a build error because `kube 1.x` depends on `k8s-openapi 0.25` but I initially specified `0.24` with `features = ["latest"]`. The `latest` feature only exists on 0.24, and having two versions of k8s-openapi in the dependency tree causes the build script to panic. Fixed by using `k8s-openapi 0.25` with an explicit `features = ["v1_32"]`.

## Observability Tools

The Kubernetes API gives us infrastructure state, but the cluster has a full monitoring stack — Prometheus, Loki, Alertmanager, and cert-manager — with their own APIs. Added `reqwest` for HTTP client calls to these in-cluster services.

### cert-manager Certificates

cert-manager CRDs aren't in `k8s-openapi` (they're custom resources), so I used kube's dynamic API:

```rust
let certs_api = kube::Api::<kube::api::DynamicObject>::all_with(
    client.clone(),
    &kube::discovery::ApiResource {
        group: "cert-manager.io".into(),
        version: "v1".into(),
        api_version: "cert-manager.io/v1".into(),
        kind: "Certificate".into(),
        plural: "certificates".into(),
    },
);
```

Returns every Certificate resource with its ready status, expiry time, renewal time, issuer, and DNS names. Now I can ask "are any certs about to expire?" and get a real answer.

### Alertmanager Alerts

Simple REST call to the Alertmanager v2 API:

```
http://monitoring-kube-prometheus-alertmanager.monitoring.svc:9093/api/v2/alerts
```

Returns active alerts with severity, status, summary, and description. If something's on fire, this is how I'll know.

### Loki Log Queries

LogQL queries against Loki:

```
http://monitoring-loki.monitoring.svc.cluster.local:3100/loki/api/v1/query_range
```

Accepts arbitrary LogQL — `{namespace="forgejo"} |= "error"`, `{job="systemd-journal"} |= "OOM"`, whatever. Returns up to 500 log lines with timestamps and stream labels. This is the "what just happened" tool.

### Prometheus Metrics

PromQL queries against Prometheus:

```
http://monitoring-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090/api/v1/query
```

Instant queries for any metric — `up`, `node_memory_MemAvailable_bytes`, `rate(container_cpu_usage_seconds_total[5m])`. This is the "how's the cluster doing right now" tool.

## Build Time

The binary got noticeably bigger with `kube-rs`, `k8s-openapi`, `reqwest`, `rustls`, and all their transitive dependencies. Build time went from ~6 minutes to ~9 minutes in the Forgejo Actions pipeline. Still native compilation on the Pi (ARM64 on ARM64), but Rust release builds with a heavy dependency tree and musl static linking are just slow on a Raspberry Pi. It is what it is.

Also bumped the pod memory limits from 64Mi to 128Mi — the TLS stack in kube-rs and reqwest needs more headroom than a bare HTTP server.

## The Full Tool Set

After this work, the MCP server exposes 11 tools:

| Tool | Source | What it does |
|------|--------|-------------|
| `get_version` | Static | Server version and build SHA |
| `get_node_status` | K8s API | Node readiness, CPU, memory, OS |
| `get_pods` | K8s API | Pod phase, restarts, node placement |
| `get_namespaces` | K8s API | Namespace listing |
| `get_events` | K8s API | Recent cluster events |
| `get_workloads` | K8s API | Deployment/StatefulSet/DaemonSet replicas |
| `get_certificates` | K8s API | cert-manager certificate status and expiry |
| `get_alerts` | Alertmanager | Active alerts with severity |
| `query_logs` | Loki | LogQL log search |
| `query_metrics` | Prometheus | PromQL metric queries |

The server went from "hello world" to "full cluster observability" in two deploys. The next obvious additions are Flux reconciliation status (is everything in sync?) and maybe ntfy notification history. But this is already enough to do a real cluster health check without touching `kubectl`.
