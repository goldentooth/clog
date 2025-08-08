# Prometheus Slurm Exporter

## Overview

Following the Slurm refactoring work, the next logical step was to add comprehensive monitoring for the HPC workload manager. This chapter documents the implementation of prometheus-slurm-exporter to provide real-time visibility into cluster utilization, job queues, and resource allocation.

## The Challenge

While Slurm was operational with 9 nodes in idle state, there was no integration with the existing Prometheus/Grafana observability stack. Key missing capabilities:

- **No Cluster Metrics**: Unable to monitor CPU/memory utilization across nodes
- **No Job Visibility**: No insight into job queues, completion rates, or resource consumption
- **No Historical Data**: No way to track cluster usage patterns over time
- **Limited Alerting**: No proactive monitoring of cluster health or resource exhaustion

## Implementation Approach

### Exporter Selection

Initially attempted the original `vpenso/prometheus-slurm-exporter` but discovered it was unmaintained and lacked modern features. Switched to the `rivosinc/prometheus-slurm-exporter` fork which provided:

- **Active Maintenance**: 87 commits, regular releases through v1.6.10
- **Pre-built Binaries**: ARM64 support via GitHub releases
- **Enhanced Features**: Job tracing, CLI fallback modes, throttling support
- **Better Performance**: Optimized for multiple Prometheus instances

### Architecture Design

Deployed the exporter following goldentooth cluster patterns:

```yaml
# Deployment Strategy
Target Nodes: slurm_controller (bettley, cargyll, dalt)
Service Port: 9092 (HTTP)
Protocol: HTTP with Prometheus file-based service discovery
Integration: Full Step-CA certificate management ready
User Management: Dedicated slurm-exporter service user
```

### Role Structure

Created `goldentooth.setup_slurm_exporter` following established conventions:

```
roles/goldentooth.setup_slurm_exporter/
├── CLAUDE.md              # Comprehensive documentation
├── tasks/main.yaml         # Main deployment tasks
├── templates/
│   ├── slurm-exporter.service.j2         # Systemd service
│   ├── slurm_targets.yaml.j2             # Prometheus targets
│   └── cert-renewer@slurm-exporter.conf.j2  # Certificate renewal
└── handlers/main.yaml      # Service management handlers
```

## Technical Implementation

### Binary Installation

```yaml
- name: 'Download prometheus-slurm-exporter from rivosinc fork'
  ansible.builtin.get_url:
    url: 'https://github.com/rivosinc/prometheus-slurm-exporter/releases/download/v{{ prometheus_slurm_exporter.version }}/prometheus-slurm-exporter_linux_{{ host.architecture }}.tar.gz'
    dest: '/tmp/prometheus-slurm-exporter-{{ prometheus_slurm_exporter.version }}.tar.gz'
    mode: '0644'
```

### Service Configuration

```ini
[Service]
Type=simple
User=slurm-exporter
Group=slurm-exporter
ExecStart=/usr/local/bin/prometheus-slurm-exporter \
  -web.listen-address={{ ansible_default_ipv4.address }}:{{ prometheus_slurm_exporter.port }} \
  -web.log-level=info
```

### Prometheus Integration

Added to the existing scrape configuration:

```yaml
prometheus_scrape_configs:
  - job_name: 'slurm'
    file_sd_configs:
      - files:
          - "/etc/prometheus/file_sd/slurm_targets.yaml"
    relabel_configs:
      - source_labels: [instance]
        target_label: instance
        regex: '([^:]+):\d+'
        replacement: '${1}'
```

### Service Discovery

Dynamic target generation for all controller nodes:

```yaml
- targets:
  - "{{ hostvars[slurm_controller].ansible_default_ipv4.address }}:{{ prometheus_slurm_exporter.port }}"
  labels:
    job: 'slurm'
    instance: '{{ slurm_controller }}'
    cluster: '{{ cluster_name }}'
    role: 'slurm-controller'
```

## Metrics Exposed

The rivosinc exporter provides comprehensive cluster visibility:

### Core Cluster Metrics
```
slurm_cpus_total 36          # Total CPU cores (9 nodes × 4 cores)
slurm_cpus_idle 36           # Available CPU cores
slurm_cpus_per_state{state="idle"} 36
slurm_node_count_per_state{state="idle"} 9
```

### Memory Utilization
```
slurm_mem_real 7.0281e+10    # Total cluster memory (MB)
slurm_mem_alloc 6.0797e+10   # Allocated memory
slurm_mem_free 9.484e+09     # Available memory
```

### Job Queue Metrics
```
slurm_jobs_pending 0         # Jobs waiting in queue
slurm_jobs_running 0         # Currently executing jobs
slurm_job_scrape_duration 29 # Metric collection performance
```

### Performance Monitoring
```
slurm_cpu_load 5.83          # Current CPU load average
slurm_node_scrape_duration 35 # Node data collection time
```

## Deployment Results

### Service Health
All three controller nodes running successfully:
```bash
● slurm-exporter.service - Prometheus Slurm Exporter
     Loaded: loaded (/etc/systemd/system/slurm-exporter.service; enabled)
     Active: active (running)
   Main PID: 3692156 (prometheus-slur)
      Tasks: 5 (limit: 8737)
     Memory: 1.5M (max: 128.0M available)
```

### Metrics Validation
```bash
curl http://10.4.0.11:9092/metrics | grep '^slurm_'
slurm_cpu_load 5.83
slurm_cpus_idle 36
slurm_cpus_per_state{state="idle"} 36
slurm_cpus_total 36
slurm_node_count_per_state{state="idle"} 9
```

### Prometheus Integration
Targets automatically discovered and scraped:
- **bettley:9092** - Controller node metrics
- **cargyll:9092** - Controller node metrics
- **dalt:9092** - Controller node metrics

## Configuration Management

### Variables Structure
```yaml
# Prometheus Slurm Exporter configuration (rivosinc fork)
prometheus_slurm_exporter:
  version: "1.6.10"
  port: 9092
  user: "slurm-exporter"
  group: "slurm-exporter"
```

### Command Interface
```bash
# Deploy exporter
goldentooth setup_slurm_exporter

# Verify deployment
goldentooth command slurm_controller "systemctl status slurm-exporter"

# Check metrics
goldentooth command bettley "curl -s http://localhost:9092/metrics | head -10"
```

## Troubleshooting Lessons

### Initial Issues Encountered

1. **Wrong Repository**: Started with unmaintained vpenso fork
   - **Solution**: Switched to actively maintained rivosinc fork

2. **TLS Configuration**: Attempted HTTPS but exporter doesn't support TLS flags
   - **Solution**: Used HTTP with plans for future TLS proxy if needed

3. **Binary Availability**: No pre-built ARM64 binaries in original version
   - **Solution**: rivosinc fork provides comprehensive release assets

4. **Port Conflicts**: Initially used port 8080
   - **Solution**: Used exporter default 9092 to avoid conflicts

### Debugging Process

Service logs were key to identifying configuration issues:
```bash
journalctl -u slurm-exporter --no-pager -l
```

Metrics endpoint testing confirmed functionality:
```bash
curl -s http://localhost:9092/metrics | grep -E '^slurm_'
```

## Integration with Existing Stack

The exporter seamlessly integrates with goldentooth monitoring infrastructure:

### Prometheus Configuration
- **File-based Service Discovery**: Automatic target management
- **Label Strategy**: Consistent with existing exporters
- **Scrape Intervals**: Standard 60-second collection

### Certificate Management
- **Step-CA Ready**: Templates prepared for future TLS implementation
- **Automatic Renewal**: Systemd timer configuration included
- **Service User**: Dedicated account with minimal permissions

### Observability Pipeline
- **Prometheus**: Metrics collection and storage
- **Grafana**: Dashboard visualization (ready for implementation)
- **Alerting**: Rule definition for cluster health monitoring

## Performance Impact

### Resource Usage
- **Memory**: ~1.5MB RSS per exporter instance
- **CPU**: Minimal impact during scraping
- **Network**: Standard HTTP metrics collection
- **Slurm Load**: Read-only operations with built-in throttling

### Scalability Considerations
- **Multiple Controllers**: Distributed across all controller nodes
- **High Availability**: No single point of failure
- **Data Consistency**: Each exporter provides complete cluster view
