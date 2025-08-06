# Certificate Renewal Debugging Odyssey

Some time after [setting up the certificate renewal system](./048_tls_cert_renewal.md), the cluster was humming along nicely with 24-hour certificate lifetimes and automatic renewal every 5 minutes. Or so I thought.

One morning, I discovered that Vault certificates had mysteriously expired overnight, despite the renewal system supposedly working. This kicked off a multi-day investigation that would lead to significant improvements in our certificate management and monitoring infrastructure.

## The Mystery: Why Didn't Vault Certificates Renew?

The first clue was puzzling - some services had renewed their certificates successfully (Consul, Nomad), while others (Vault) had failed silently. The cert-renewer systemd service showed no errors, and the timers were running on schedule.

```bash
$ goldentooth command_root jast 'systemctl status cert-renewer@vault.timer'
● cert-renewer@vault.timer - Timer for certificate renewal of vault
     Loaded: loaded (/etc/systemd/system/cert-renewer@.timer; enabled)
     Active: active (waiting) since Wed 2025-07-23 14:05:12 EDT; 3h ago
```

The timer was active, but the certificates were still expired. Something was fundamentally wrong with our renewal logic.

## Building a Certificate Renewal Canary

Rather than guessing at the problem, I decided to build proper test infrastructure. The solution was a "canary" service - a minimal certificate renewal setup with extremely short lifetimes that would fail fast and give us rapid feedback.

### Creating the Canary Service

I created a new Ansible role `goldentooth.setup_cert_renewer_canary` that:

1. **Creates a dedicated user and service**: `cert-canary` user with its own systemd service
2. **Uses 15-minute certificate lifetimes**: Fast enough to debug quickly
3. **Runs on a 5-minute timer**: Same schedule as production services
4. **Provides comprehensive logging**: Detailed output for debugging

```yaml
# roles/goldentooth.setup_cert_renewer_canary/defaults/main.yaml
cert_canary:
  username: cert-canary
  group: cert-canary
  cert_lifetime: 15m
  cert_path: /opt/cert-canary/certs/tls.crt
  key_path: /opt/cert-canary/certs/tls.key
```

The canary service template includes detailed logging:

```systemd
[Unit]
Description=Certificate Canary Service
After=network-online.target

[Service]
Type=oneshot
User=cert-canary
WorkingDirectory=/opt/cert-canary
ExecStart=/bin/echo "Certificate canary service executed successfully"
```

## Discovering the Root Cause

With the canary in place, I could observe the renewal process in real-time. The breakthrough came when I examined the `step certificate needs-renewal` command more carefully.

### The 66% Threshold Problem

The default cert-renewer configuration uses a 66% threshold for renewal - certificates renew when they have less than 66% of their lifetime remaining. For 24-hour certificates, this means renewal occurs when there are about 8 hours left.

But here's the critical issue: with a 5-minute timer interval, there's only a narrow window for successful renewal. If the renewal fails during that window (due to network issues, service restarts, etc.), the next attempt won't occur until the timer fires again.

The math was unforgiving:
- **24-hour certificate**: 66% threshold = ~8 hour renewal window
- **5-minute timer**: 12 attempts per hour
- **Network/service instability**: Occasional renewal failures
- **Result**: Certificates could expire if multiple renewal attempts failed in succession

## The Solution: Environment Variable Configuration

The fix involved making the cert-renewer system more configurable and robust. I updated the base `cert-renewer@.service` template to support environment variable overrides:

```systemd
[Unit]
Description=Certificate renewer for %I
After=network-online.target
Documentation=https://smallstep.com/docs/step-ca/certificate-authority-server-production
StartLimitIntervalSec=0

[Service]
Type=oneshot
User=root
Environment=STEPPATH=/etc/step-ca
Environment=CERT_LOCATION=/etc/step/certs/%i.crt
Environment=KEY_LOCATION=/etc/step/certs/%i.key
Environment=EXPIRES_IN_THRESHOLD=66%

ExecCondition=/usr/bin/step certificate needs-renewal ${CERT_LOCATION} --expires-in ${EXPIRES_IN_THRESHOLD}
ExecStart=/usr/bin/step ca renew --force ${CERT_LOCATION} ${KEY_LOCATION}
ExecStartPost=/usr/bin/env sh -c "! systemctl --quiet is-active %i.service || systemctl try-reload-or-restart %i.service"

[Install]
WantedBy=multi-user.target
```

### Service-Specific Overrides

Each service now gets its own override configuration that specifies the exact certificate paths and renewal behavior:

```systemd
# /etc/systemd/system/cert-renewer@vault.service.d/override.conf
[Service]
Environment=CERT_LOCATION=/opt/vault/tls/tls.crt
Environment=KEY_LOCATION=/opt/vault/tls/tls.key
WorkingDirectory=/opt/vault/tls

ExecStartPost=/usr/bin/env sh -c "! systemctl --quiet is-active vault.service || systemctl try-reload-or-restart vault.service"
```

The beauty of this approach is that we can now tune renewal behavior per service without modifying the base template.

## Comprehensive Monitoring Infrastructure

While debugging the certificate issue, I also built comprehensive monitoring dashboards and alerting to prevent future incidents.

### New Grafana Dashboards

I created three major monitoring dashboards:

1. **Slurm Cluster Overview**: Job queue metrics, resource utilization, historical trends
2. **HashiCorp Services Overview**: Consul health, Vault status, Nomad allocation monitoring
3. **Infrastructure Health Overview**: Node uptime, storage capacity, network metrics

### Enhanced Metrics Collection

The monitoring improvements included:

- **Vector Internal Metrics**: Enabled Vector's internal metrics and Prometheus exporter
- **Certificate Expiration Tracking**: Automated monitoring of certificate days-remaining
- **Service Health Indicators**: Real-time status for all critical cluster services
- **Alert Rules**: Proactive notifications for certificate expiration and service failures

## Testing Infrastructure Improvements

The certificate renewal investigation led to significant improvements in our testing infrastructure.

### Certificate-Aware Test Suite

I created a comprehensive `test_certificate_renewal` role that:

1. **Node-Specific Testing**: Only tests certificates for services actually deployed on each node
2. **Multi-Layered Validation**: Certificate presence, validity, timer status, renewal capability
3. **Chain Validation**: Verifies certificates against the cluster CA
4. **Canary Health Monitoring**: Tracks the certificate canary's renewal cycles

### Smart Service Filtering

The test improvements included intelligent service filtering:

```yaml
# Filter services to only those deployed on this node
- name: Filter services for current node
  set_fact:
    node_certificate_services: |-
      {%- set filtered_services = [] -%}
      {%- for service in certificate_services -%}
        {%- set should_include = false -%}
        {%- if service.get('specific_hosts') -%}
          {%- if inventory_hostname in service.specific_hosts -%}
            {%- set should_include = true -%}
          {%- endif -%}
        {%- elif service.host_groups -%}
          {%- for group in service.host_groups -%}
            {%- if inventory_hostname in groups.get(group, []) -%}
              {%- set should_include = true -%}
            {%- endif -%}
          {%- endfor -%}
        {%- endif -%}
        {%- if should_include -%}
          {%- set _ = filtered_services.append(service) -%}
        {%- endif -%}
      {%- endfor -%}
      {{ filtered_services }}
```

This eliminated false positives where tests were failing for missing certificates on nodes where services weren't supposed to be running.
