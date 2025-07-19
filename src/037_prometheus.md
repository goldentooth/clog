# Prometheus

Way back in [Chapter 19](./019_prometheus_node_exporter.md), I set up an Prometheus Node Exporter "app" for Argo CD, but I never actually set up Prometheus itself.

That's really fairly odd for me, since I'm normally super twitchy about metrics, logging, and observability. I guess I put it off because I was dealing with some kind of existential questions; where would Prometheus live, how would it communicate, etc, but then ended up kinda running out of steam before I answered the questions.

So, better late than never, I'm going to work on setting up Prometheus in a nice, decentralized kind of way.

## Implementation Architecture

### Installation Method

I'm using the official [prometheus.prometheus.prometheus](https://prometheus-community.github.io/ansible/branch/main/prometheus_role.html) Ansible role from the Prometheus community. The depth to Prometheus is, after all, configuring and using it, not merely in installing it.

The installation is managed through:
- **Playbook**: `setup_prometheus.yaml`
- **Custom role**: `goldentooth.setup_prometheus` (wraps the community role)
- **CLI command**: `goldentooth setup_prometheus`

### Deployment Location

Prometheus runs on `allyrion` (10.4.0.10), which consolidates multiple infrastructure services:
- HAProxy load balancer
- NFS server
- Prometheus monitoring server

This placement provides several advantages:
- Central location for cluster-wide monitoring
- Proximity to load balancer for HAProxy metrics
- Reduced resource usage on Kubernetes worker nodes

## Service Configuration

### Core Settings

The Prometheus service is configured with production-ready settings:

```yaml
# Storage and retention
prometheus_storage_retention_time: "15d"
prometheus_storage_retention_size: "5GB"
prometheus_storage_tsdb_path: "/var/lib/prometheus"

# Network and performance
prometheus_web_listen_address: "0.0.0.0:9090"
prometheus_config_global_scrape_interval: "60s"
prometheus_config_global_evaluation_interval: "15s"
prometheus_config_global_scrape_timeout: "15s"
```

### Security Hardening

The service implements comprehensive security measures:

- **Dedicated user**: Runs as `prometheus` user/group
- **Systemd hardening**: NoNewPrivileges, PrivateDevices, ProtectSystem=strict
- **Capability restrictions**: Limited to CAP_SET_UID only
- **Resource limits**: GOMAXPROCS=4 to prevent CPU exhaustion

### External Labels

Cluster identification through external labels:

```yaml
external_labels:
  environment: goldentooth
  cluster: goldentooth
  domain: goldentooth.net
```

## Service Discovery Implementation

### File-Based Service Discovery

Rather than relying on complex auto-discovery, I implement file-based service discovery for reliability and explicit control:

**Target Generation** (`/etc/prometheus/file_sd/node.yaml`):
```yaml
{% for host in groups['all'] %}
- targets:
    - "{{ hostvars[host]['ipv4_address'] }}:9100"
  labels:
    instance: "{{ host }}"
    job: 'node'
{% endfor %}
```

This approach:
- Auto-generates targets from Ansible inventory
- Covers all 13 cluster nodes (12 Raspberry Pis + 1 x86 GPU node)
- Provides consistent labeling with `instance` and `job` labels
- Updates automatically when nodes are added/removed

## Scrape Configurations

### Core Infrastructure Monitoring

**Prometheus Self-Monitoring**:
```yaml
- job_name: 'prometheus'
  static_configs:
    - targets: ['allyrion:9090']
```

**HAProxy Load Balancer**:
```yaml
- job_name: 'haproxy'
  static_configs:
    - targets: ['allyrion:8405']
```

HAProxy includes a built-in Prometheus exporter accessible at `/metrics` on port 8405, providing load balancer performance and health metrics.

**Nginx Reverse Proxy**:
```yaml
- job_name: 'nginx'
  static_configs:
    - targets: ['allyrion:9113']
```

### Node Monitoring

**File Service Discovery** for all cluster nodes:
```yaml
- job_name: "unknown"
  file_sd_configs:
    - files:
      - "/etc/prometheus/file_sd/*.yaml"
      - "/etc/prometheus/file_sd/*.json"
```

This targets all Node Exporter instances across the cluster, providing comprehensive infrastructure metrics.

### Advanced Integrations

**Loki Log Aggregation**:
```yaml
- job_name: 'loki'
  static_configs:
    - targets: ['inchfield:3100']
  scheme: 'https'
  tls_config:
    ca_file: /etc/ssl/certs/goldentooth.pem
```

This integration uses the Step-CA certificate authority for secure communication with the Loki log aggregation service.

## Exporter Ecosystem

### Node Exporter Deployment

**Kubernetes Nodes** (via Argo CD):
- **Helm Chart**: prometheus-node-exporter v4.46.1  
- **Namespace**: prometheus-node-exporter
- **Extra Collectors**: `--collector.systemd`, `--collector.processes`
- **Management**: Automated GitOps deployment with auto-sync

**Infrastructure Node** (allyrion):
- **Installation**: Via `prometheus.prometheus.node_exporter` role
- **Enabled Collectors**: systemd for service monitoring
- **Integration**: Direct scraping by local Prometheus instance

### Application Exporters

I also configured several application-specific exporters:

**HAProxy Built-in Exporter**: Provides load balancer metrics including backend health, response times, and traffic distribution

**Nginx Exporter**: Monitors reverse proxy performance and request patterns

## Network Access and Security

### Nginx Reverse Proxy

To provide secure external access to Prometheus, I configured an Nginx reverse proxy:

```nginx
server {
    listen 8081;
    location / {
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Application prometheus;
    }
}
```

This provides:
- Network isolation (Prometheus only accessible locally)
- Header injection for request identification
- Potential for future authentication layer

### Certificate Integration

The cluster uses Step-CA for comprehensive certificate management. Prometheus leverages this infrastructure for:
- Secure scraping of TLS-enabled services (Loki)
- Potential future TLS termination
- Integration with the broader security model

## Alerting Configuration

### Basic Alert Rules

The installation includes foundational alerting rules in `/etc/prometheus/rules/ansible_managed.yml`:

**Watchdog Alert**: Always-firing alert to verify the alerting pipeline is functional

**Instance Down Alert**: Critical alert when `up == 0` for 5 minutes, indicating node or service failure

### Future Expansion

The alert rule framework is prepared for expansion with application-specific alerts, SLA monitoring, and capacity planning alerts.

## Integration with Monitoring Stack

### Grafana Integration

Prometheus serves as the primary data source for Grafana dashboards:

```yaml
datasources:
  - name: prometheus
    type: prometheus  
    url: http://allyrion:9090
    access: proxy
```

This enables rich visualization of cluster metrics through pre-configured and custom dashboards.

### Storage and Performance

**TSDB Configuration**:
- **Retention**: 15 days (time) and 5GB (size) for appropriate data lifecycle
- **Storage**: Local disk at `/var/lib/prometheus`
- **Compaction**: Automatic TSDB compaction for optimal query performance

The scrape configuration was fairly straightforward, and the result is a comprehensive monitoring foundation covering all infrastructure components and preparing for future application-specific monitoring expansion.
