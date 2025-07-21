# Comprehensive Metrics Collection

After establishing the foundation of our observability stack with Prometheus, Grafana, and the blackbox exporter, it's time to ensure we're collecting metrics from every critical component in our cluster. This chapter covers the addition of Nomad telemetry and Kubernetes object metrics to our monitoring infrastructure.

## The Metrics Audit

A comprehensive audit of our cluster revealed which services were already exposing metrics:

**Already Configured:**
- ✅ Kubernetes API server, controller manager, scheduler (via control plane endpoints)
- ✅ HAProxy (custom exporter on port 8405)
- ✅ Prometheus (self-monitoring)
- ✅ Grafana (internal metrics)
- ✅ Loki (log aggregation metrics)
- ✅ Consul (built-in Prometheus endpoint)
- ✅ Vault (telemetry endpoint)

**Missing:**
- ❌ Nomad (no telemetry configuration)
- ❌ Kubernetes object state (deployments, pods, services)

## Enabling Nomad Telemetry

Nomad has built-in Prometheus support but requires explicit configuration. We added the telemetry block to our Nomad configuration template:

```hcl
telemetry {
  collection_interval        = "1s"
  disable_hostname           = true
  prometheus_metrics         = true
  publish_allocation_metrics = true
  publish_node_metrics       = true
}
```

This configuration:
- Enables Prometheus-compatible metrics on `/v1/metrics?format=prometheus`
- Publishes detailed allocation and node metrics
- Disables hostname labels (we add our own)
- Sets a 1-second collection interval for fine-grained data

## Certificate-Based Authentication

Unlike some services that expose metrics without authentication, Nomad requires mutual TLS for metrics access. We leveraged our Step-CA infrastructure to generate proper client certificates:

```yaml
- name: 'Generate Prometheus client certificate for Nomad metrics.'
  ansible.builtin.shell:
    cmd: |
      {{ step_ca.executable }} ca certificate \
        "prometheus.client.nomad" \
        "/etc/prometheus/certs/nomad-client.crt" \
        "/etc/prometheus/certs/nomad-client.key" \
        --provisioner="{{ step_ca.default_provisioner.name }}" \
        --password-file="{{ step_ca.default_provisioner.password_path }}" \
        --san="prometheus.client.nomad" \
        --san="prometheus" \
        --san="{{ clean_hostname }}" \
        --san="{{ ipv4_address }}" \
        --not-after='24h' \
        --console \
        --force
```

This approach ensures:
- Certificates are properly signed by our cluster CA
- Client identity is clearly established
- Automatic renewal via systemd timers
- Consistent with our security model

## Prometheus Scrape Configuration

With certificates in place, we configured Prometheus to scrape all Nomad nodes:

```yaml
- job_name: 'nomad'
  metrics_path: '/v1/metrics'
  params:
    format: ['prometheus']
  static_configs:
    - targets:
        - "10.4.0.11:4646"  # bettley (server)
        - "10.4.0.12:4646"  # cargyll (server)
        - "10.4.0.13:4646"  # dalt (server)
        # ... all client nodes
  scheme: 'https'
  tls_config:
    ca_file: "{{ step_ca.root_cert_path }}"
    cert_file: "/etc/prometheus/certs/nomad-client.crt"
    key_file: "/etc/prometheus/certs/nomad-client.key"
```

## Kubernetes Object Metrics with kube-state-metrics

While node-level metrics tell us about resource usage, we also need visibility into Kubernetes objects themselves. Enter kube-state-metrics, which exposes metrics about:

- Deployment replica counts and rollout status
- Pod phases and container states
- Service endpoints and readiness
- PersistentVolume claims and capacity
- Job completion status
- And much more

## GitOps Deployment Pattern

Following our established patterns, we created a dedicated GitOps repository for kube-state-metrics:

```bash
# Create the repository
gh repo create goldentooth/kube-state-metrics --public

# Clone into our organization structure
cd ~/Projects/goldentooth
git clone https://github.com/goldentooth/kube-state-metrics.git

# Add the required label for Argo CD discovery
gh repo edit goldentooth/kube-state-metrics --add-topic gitops-repo
```

The key insight here is that our Argo CD ApplicationSet automatically discovers repositories with the `gitops-repo` label, eliminating manual application creation.

## kube-state-metrics Configuration

The deployment includes comprehensive RBAC permissions to read all Kubernetes objects:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-state-metrics
rules:
- apiGroups: [""]
  resources:
  - configmaps
  - secrets
  - nodes
  - pods
  - services
  - resourcequotas
  - replicationcontrollers
  - limitranges
  - persistentvolumeclaims
  - persistentvolumes
  - namespaces
  - endpoints
  verbs: ["list", "watch"]
# ... additional resources
```

We discovered that some resources like `resourcequotas`, `replicationcontrollers`, and `limitranges` were missing from the initial configuration, causing permission errors. A quick update to the ClusterRole resolved these issues.

## Security Hardening

The kube-state-metrics deployment follows security best practices:

```yaml
securityContext:
  fsGroup: 65534
  runAsGroup: 65534
  runAsNonRoot: true
  runAsUser: 65534
  seccompProfile:
    type: RuntimeDefault
```

Container-level security adds additional restrictions:

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true
```

## Prometheus Auto-Discovery

The service includes annotations for automatic Prometheus discovery:

```yaml
annotations:
  prometheus.io/scrape: 'true'
  prometheus.io/port: '8080'
  prometheus.io/path: '/metrics'
```

This eliminates the need for manual Prometheus configuration - the metrics are automatically discovered and scraped.

## Verifying the Deployment

After deployment, we can verify metrics are being exposed:

```bash
# Port-forward to test locally
kubectl port-forward -n kube-state-metrics service/kube-state-metrics 8080:8080

# Check deployment metrics
curl -s http://localhost:8080/metrics | grep kube_deployment_status_replicas
```

Example output:
```
kube_deployment_status_replicas{namespace="argocd",deployment="argocd-redis-ha-haproxy"} 3
kube_deployment_status_replicas{namespace="kube-system",deployment="coredns"} 2
```
