# Prometheus Blackbox Exporter

## The Observability Gap

Our Goldentooth cluster has comprehensive infrastructure monitoring through Prometheus, node exporters, and application metrics. But we've been missing a crucial piece: **synthetic monitoring**. We can see if our servers are running, but can we actually reach our services? Are our web UIs accessible? Can we connect to our APIs?

Enter the Prometheus Blackbox Exporter - our eyes and ears for service availability across the entire cluster.

## What is Blackbox Monitoring?

Blackbox monitoring tests services from the outside, just like your users would. Instead of looking at internal metrics, it:

- **Probes HTTP/HTTPS endpoints** - "Is the Consul web UI actually working?"
- **Tests TCP connectivity** - "Can I connect to the Vault API port?"
- **Validates DNS resolution** - "Do our cluster domains resolve correctly?"
- **Checks ICMP reachability** - "Are all nodes responding to ping?"

It's called "blackbox" because we don't peek inside the service - we just test if it works from the outside.

## Planning the Implementation

I needed to design a comprehensive monitoring strategy that would cover:

### Service Categories
- **HashiCorp Stack**: Consul, Vault, Nomad web UIs and APIs
- **Kubernetes Services**: API server health, Argo CD, LoadBalancer services
- **Observability Stack**: Prometheus, Grafana, Loki endpoints
- **Infrastructure**: All 13 node homepages, HAProxy stats
- **External Services**: CloudFront distributions
- **Network Health**: DNS resolution for all cluster domains

### Intelligent Probe Types
- **Internal HTTPS**: Uses Step-CA certificates for cluster services
- **External HTTPS**: Uses public CAs for external services
- **HTTP**: Plain HTTP for internal services
- **TCP**: Port connectivity for APIs and cluster communication
- **DNS**: Domain resolution for cluster services
- **ICMP**: Basic network connectivity for all nodes

## The Ansible Implementation

I created a comprehensive Ansible role `goldentooth.setup_blackbox_exporter` that handles:

### Core Deployment
```yaml
# Install blackbox exporter v0.25.0
# Deploy on allyrion (same node as Prometheus)
# Configure systemd service with security hardening
# Set up TLS certificates via Step-CA
```

### Security Integration
The blackbox exporter integrates seamlessly with our Step-CA infrastructure:
- **Client certificates** for secure communication
- **CA validation** for internal services
- **Automatic renewal** via systemd timers
- **Proper certificate ownership** for the service user

### Service Discovery Magic
Instead of hardcoding targets, I implemented dynamic service discovery:

```yaml
# Generate targets from Ansible inventory variables
blackbox_https_internal_targets:
  - "https://consul.goldentooth.net:8501"
  - "https://vault.goldentooth.net:8200"
  - "https://nomad.goldentooth.net:4646"
  # ... and many more

# Auto-generate ICMP targets for all cluster nodes
{% for host in groups['all'] %}
- targets:
    - "{{ hostvars[host]['ipv4_address'] }}"
  labels:
    job: 'blackbox-icmp'
    node: "{{ host }}"
{% endfor %}
```

### Prometheus Integration
The trickiest part was configuring Prometheus to properly scrape blackbox targets. Blackbox exporter works differently than normal exporters:

```yaml
# Instead of scraping the target directly...
# Prometheus scrapes the blackbox exporter with target as parameter
- job_name: 'blackbox-https-internal'
  metrics_path: '/probe'
  params:
    module: ['https_2xx_internal']
  relabel_configs:
    # Redirect to blackbox exporter
    - target_label: __address__
      replacement: "allyrion:9115"
    # Pass original target as parameter
    - source_labels: [__param_target]
      target_label: __param_target
```

## Deployment Day

The deployment was mostly smooth with a few interesting challenges:

### Certificate Duration Drama
```bash
# First attempt failed:
# "requested duration of 8760h is more than authorized maximum of 168h"

# Solution: Match Step-CA policy
--not-after=168h  # 1 week instead of 1 year
```

### DNS Resolution Reality Check
Many of our internal domains (`*.goldentooth.net`) don't actually resolve yet, so probes show `up=0`. This is expected and actually valuable - it shows us what infrastructure we still need to set up!

### Relabel Configuration Complexity
Getting the Prometheus relabel configs right for blackbox took several iterations. The key insight: blackbox exporter targets need to be "redirected" through the exporter itself.

## What We're Monitoring Now

The blackbox exporter is now actively monitoring **40+ endpoints** across our cluster:

### Web UIs and APIs
- Consul Web UI (`https://consul.goldentooth.net:8501`)
- Vault Web UI (`https://vault.goldentooth.net:8200`)
- Nomad Web UI (`https://nomad.goldentooth.net:4646`)
- Grafana Dashboard (`https://grafana.goldentooth.net:3000`)
- Argo CD Interface (`https://argocd.goldentooth.net`)

### Infrastructure Endpoints
- All 13 node homepages (`http://[node].nodes.goldentooth.net`)
- HAProxy statistics page (with basic auth)
- Prometheus web interface
- Loki API endpoints

### Network Connectivity
- TCP connectivity to all critical service ports
- DNS resolution for all cluster domains
- ICMP ping for every cluster node
- External CloudFront distributions

## The Power of Synthetic Monitoring

Now when something breaks, we'll know immediately:
- **`probe_success`** tells us if the service is reachable
- **`probe_duration_seconds`** shows response times
- **`probe_http_status_code`** reveals HTTP errors
- **`probe_ssl_earliest_cert_expiry`** warns about certificate expiration

This complements our existing infrastructure monitoring perfectly. We can see both "the server is running" (node exporter) and "the service actually works" (blackbox exporter).
