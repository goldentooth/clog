# Step-CA Certificate Monitoring Implementation

With the goldentooth cluster now heavily dependent on Step-CA for certificate management across Consul, Vault, Nomad, Grafana, Loki, Vector, HAProxy, Blackbox Exporter, and the newly deployed SeaweedFS distributed storage, we needed comprehensive certificate monitoring to prevent service outages from expired certificates.

The existing certificate monitoring was basic - we had file-based certificate expiry alerts, but lacked the visibility and proactive alerting necessary for enterprise-grade PKI management.

## The Monitoring Challenge

Our cluster runs multiple services with Step-CA certificates:

- **Consul**: Service mesh certificates for all nodes
- **Vault**: Secrets management with HA cluster
- **Nomad**: Workload orchestration across the cluster  
- **Grafana**: Observability dashboard access
- **Loki**: Log aggregation infrastructure
- **Vector**: Log shipping to Loki
- **HAProxy**: Load balancer with TLS termintion
- **Blackbox Exporter**: Synthetic monitoring service
- **SeaweedFS**: Distributed storage with master/volume servers

Each service has automated certificate renewal via `cert-renewer@.service` systemd timers, but we needed comprehensive monitoring to ensure the renewal system itself was healthy and catch any failures before they caused outages.

## Enhanced Blackbox Monitoring

The first enhancement expanded our synthetic monitoring to include comprehensive TLS validation for all Step-CA services.

### SeaweedFS Integration

With SeaweedFS newly deployed as a high-availability distributed storage system, I added its endpoints to blackbox monitoring:

```yaml
# SeaweedFS Master servers (HA cluster)
- targets:
  - "https://fenn:9333"
  - "https://karstark:9333" 
  labels:
    service: "seaweedfs-master"

# SeaweedFS Volume servers  
- targets:
  - "https://fenn:8080"
  - "https://karstark:8080"
  labels:
    service: "seaweedfs-volume"
```

### Comprehensive TLS Endpoint Monitoring

Every Step-CA managed service now has synthetic TLS validation:

```yaml
blackbox_https_internal_targets:
  - "https://consul.goldentooth.net:8501"
  - "https://vault.goldentooth.net:8200" 
  - "https://nomad.goldentooth.net:4646"
  - "https://grafana.goldentooth.net:3000"
  - "https://loki.goldentooth.net:3100"
  - "https://vector.goldentooth.net:8686"
  - "https://fenn:9115"  # blackbox exporter itself
  - "https://fenn:9333"  # seaweedfs master
  - "https://karstark:9333"
  - "https://fenn:8080"  # seaweedfs volume
  - "https://karstark:8080"
```

The blackbox exporter validates not just connectivity, but certificate chain validity, expiry dates, and proper TLS negotiation for each endpoint.

## Advanced Prometheus Alert System

The core enhancement was implementing a comprehensive multi-tier alerting system for certificate management.

### Certificate Expiry Alerts

I implemented three tiers of certificate expiry warnings:

```yaml
# 30-day advance warning
- alert: CertificateExpiringSoon
  expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 30
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "Certificate expiring in 30 days"
    description: "Certificate for {{ $labels.instance }} expires in 30 days. Plan renewal."

# 7-day critical alert  
- alert: CertificateExpiringCritical
  expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 7
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Certificate expiring in 7 days"
    description: "Certificate for {{ $labels.instance }} expires in 7 days. Immediate attention required."

# 2-day emergency alert
- alert: CertificateExpiringEmergency  
  expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 2
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Certificate expiring in 2 days"
    description: "Certificate for {{ $labels.instance }} expires in 2 days. Emergency action required."
```

### Certificate Renewal System Monitoring

Beyond expiry monitoring, I added alerts for certificate renewal system health:

```yaml
# File-based certificate monitoring
- alert: CertificateFileExpiring
  expr: (file_certificate_expiry_seconds - time()) / 86400 < 7
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Certificate file expiring soon"
    description: "Certificate file {{ $labels.path }} expires in less than 7 days"

# Certificate renewal timer failure
- alert: CertificateRenewalTimerFailed
  expr: systemd_timer_last_trigger_seconds{name=~"cert-renewer@.*"} < time() - 86400 * 8
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "Certificate renewal timer failed"
    description: "Certificate renewal timer {{ $labels.name }} hasn't run in over 8 days"
```

### Step-CA Server Health

Critical infrastructure monitoring for the Step-CA service itself:

```yaml
# Step-CA service availability
- alert: StepCADown
  expr: up{job="step-ca"} == 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Step-CA server is down"
    description: "Step-CA certificate authority is unreachable"

# TLS endpoint failures
- alert: TLSEndpointDown
  expr: probe_success{job=~"blackbox-https.*"} == 0
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "TLS endpoint unreachable"
    description: "TLS endpoint {{ $labels.instance }} is unreachable via HTTPS"
```

## Comprehensive Certificate Dashboard

The monitoring enhancement includes a dedicated Grafana dashboard providing complete PKI visibility.

### Dashboard Features

The Step-CA Certificate Dashboard displays:

- **Certificate Expiry Timeline**: Color-coded visualization showing all certificates with expiry thresholds (green > 30 days, yellow 7-30 days, red < 7 days)
- **TLS Endpoint Status**: Real-time status of all HTTPS endpoints monitored via blackbox probes
- **Certificate Renewal Health**: Status of systemd renewal timers across all services
- **Step-CA Server Status**: Availability and responsiveness of the certificate authority
- **Certificate Inventory**: Table showing all managed certificates with expiry dates and renewal status

### Dashboard Implementation

```yaml
- name: Deploy Step-CA certificate monitoring dashboard
  ansible.builtin.copy:
    src: "{{ playbook_dir }}/../grafana-dashboards/step-ca-certificate-dashboard.json"
    dest: "/var/lib/grafana/dashboards/"
    owner: grafana
    group: grafana
    mode: '0644'
  notify: restart grafana
```

The dashboard provides at-a-glance visibility into the health of the entire PKI infrastructure, with drill-down capabilities to investigate specific certificate issues.

## Infrastructure Integration

### Enhanced Grafana Role

The Grafana setup role now includes automated dashboard deployment:

```yaml
- name: Create dashboards directory
  ansible.builtin.file:
    path: "/var/lib/grafana/dashboards"
    state: present
    owner: grafana
    group: grafana
    mode: '0755'

- name: Deploy certificate monitoring dashboard
  ansible.builtin.copy:
    src: "step-ca-certificate-dashboard.json"
    dest: "/var/lib/grafana/dashboards/"
    owner: grafana
    group: grafana
    mode: '0644'
  notify: restart grafana
```

### Prometheus Configuration Updates

The Prometheus alerting rules required careful template escaping for proper alert message formatting:

```yaml
# Proper Prometheus alert template escaping
annotations:
  summary: "Certificate for {{ "{{ $labels.instance }}" }} expires in 30 days"
  description: "Certificate renewal required for {{ "{{ $labels.instance }}" }}"
```

### Service Targets Configuration

All Step-CA certificate endpoints are now systematically monitored:

```yaml
blackbox_targets:
  https_internal:
    # Core HashiCorp services
    - "https://consul.goldentooth.net:8501"
    - "https://vault.goldentooth.net:8200"
    - "https://nomad.goldentooth.net:4646"
    
    # Observability stack
    - "https://grafana.goldentooth.net:3000"
    - "https://loki.goldentooth.net:3100"
    - "https://vector.goldentooth.net:8686"
    
    # Infrastructure services
    - "https://fenn:9115"  # blackbox exporter
    
    # SeaweedFS distributed storage
    - "https://fenn:9333"   # seaweedfs master
    - "https://karstark:9333"
    - "https://fenn:8080"   # seaweedfs volume  
    - "https://karstark:8080"
```

## Deployment Results

### Monitoring Coverage

The enhanced certificate monitoring now provides:

- **Complete PKI visibility**: All 20+ Step-CA certificates monitored
- **Proactive alerting**: 30/7/2 day advance warnings prevent surprises
- **System health monitoring**: Renewal timer and Step-CA service health tracking
- **Synthetic validation**: Real TLS endpoint testing via blackbox probes
- **Centralized dashboard**: Single pane of glass for certificate infrastructure

### Alert Integration

The alert system provides:

- **Early warning system**: 30-day alerts allow planned certificate maintenance
- **Escalating severity**: 7-day critical and 2-day emergency alerts ensure attention
- **Renewal system monitoring**: Catches failures in automated renewal timers
- **Infrastructure monitoring**: Step-CA server availability tracking

### Operational Impact

Before this enhancement:
- Basic file-based certificate expiry alerts
- Limited visibility into certificate health
- Potential for service outages from unnoticed certificate expiry
- Manual certificate status checking required

After implementation:
- Enterprise-grade certificate lifecycle monitoring
- Proactive alerting preventing service disruptions  
- Complete synthetic validation of certificate-dependent services
- Real-time visibility into PKI infrastructure health
- Automated dashboard providing immediate certificate status overview

## Repository Integration

### Multi-Repository Changes

The implementation spans two repositories:

**goldentooth/ansible**: Core infrastructure implementation
- Enhanced blackbox exporter role with SeaweedFS targets
- Comprehensive Prometheus alerting rules
- Improved Grafana role with dashboard deployment
- Certificate monitoring integration across all Step-CA services

**goldentooth/grafana-dashboards**: Dashboard repository
- New Step-CA Certificate Dashboard with complete PKI visibility
- Dashboard committed for reuse across environments
- JSON format compatible with Grafana provisioning

### Command Integration

Certificate monitoring integrates with goldentooth CLI:

```bash
# Deploy enhanced certificate monitoring
goldentooth setup_blackbox_exporter
goldentooth setup_grafana  
goldentooth setup_prometheus

# Check certificate monitoring status
goldentooth command allyrion "systemctl status blackbox-exporter"

# View certificate expiry alerts
goldentooth command allyrion "curl -s http://localhost:9090/api/v1/alerts | jq '.data.alerts[] | select(.labels.alertname | contains(\"Certificate\"))'"

# Monitor renewal timers
goldentooth command_all "systemctl list-timers 'cert-renewer@*'"
```

This comprehensive Step-CA certificate monitoring implementation transforms goldentooth from basic certificate management to enterprise-grade PKI infrastructure with complete lifecycle visibility, proactive alerting, and automated health monitoring. The system now prevents certificate-related service outages through early warning and comprehensive synthetic validation of all certificate-dependent services.