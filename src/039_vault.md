# Vault

As long as I'm setting up Consul, I figure I might as well set up Vault too.

This wasn't that bad, compared to the experience I had with ACLs in Consul. I set up a KMS key for unsealing, generated a certificate authority and regenerated TLS assets for my three server nodes, and the Consul storage backend works seamlessly.

## Vault Cluster Architecture

### Deployment Configuration

The Vault cluster runs on three Raspberry Pi nodes: `bettley`, `cargyll`, and `dalt`. This provides high availability with automatic leader election and fault tolerance.

**Key Design Decisions**:
- **Storage Backend**: Consul (not Raft) - leverages existing Consul cluster for data persistence
- **Auto-Unsealing**: AWS KMS integration eliminates manual unsealing after restarts
- **TLS Everywhere**: Full mutual TLS with Step-CA integration
- **Service Integration**: Deep integration with Consul service discovery

### AWS KMS Auto-Unsealing

Rather than managing unseal keys manually, I implemented AWS KMS auto-unsealing through Terraform:

**KMS Key Configuration** (`terraform/modules/vault_seal/kms.tf`):
```hcl
resource "aws_kms_key" "vault_seal" {
  description             = "KMS key for managing the Goldentooth vault seal"
  key_usage               = "ENCRYPT_DECRYPT"
  enable_key_rotation     = true
  deletion_window_in_days = 30
}

resource "aws_kms_alias" "vault_seal" {
  name          = "alias/goldentooth/vault-seal"
  target_key_id = aws_kms_key.vault_seal.key_id
}
```

This provides:
- **Automatic unsealing** on service restart
- **Key rotation** managed by AWS
- **Audit trail** through CloudTrail
- **No manual intervention** required for cluster recovery

## Vault Server Configuration

### Core Settings

The main Vault configuration demonstrates production-ready patterns:

```hcl
ui                                  = true
cluster_addr                        = "https://{{ ipv4_address }}:8201"
api_addr                            = "https://{{ ipv4_address }}:8200"
disable_mlock                       = true
cluster_name                        = "goldentooth"
enable_response_header_raft_node_id = true
log_level                           = "debug"
```

**Key Features**:
- **Web UI enabled** for administrative access
- **Per-node cluster addressing** using individual IP addresses
- **Memory lock disabled** (appropriate for container/Pi environments)
- **Debug logging** for troubleshooting and development

### Storage Backend: Consul Integration

```hcl
storage "consul" {
  address           = "{{ ipv4_address }}:8500"
  check_timeout     = "5s"
  consistency_mode  = "strong"
  path              = "vault/"
  token             = "{{ vault_consul_token.token.SecretID }}"
}
```

The Consul storage backend provides:
- **Strong consistency** for data integrity
- **Leveraged infrastructure** - reuses existing Consul cluster
- **ACL integration** with dedicated Consul tokens
- **Service discovery** through Consul's native mechanisms

### TLS Configuration

```hcl
listener "tcp" {
  address                             = "{{ ipv4_address }}:8200"
  tls_cert_file                       = "/opt/vault/tls/tls.crt"
  tls_key_file                        = "/opt/vault/tls/tls.key"
  tls_require_and_verify_client_cert  = true
  telemetry {
    unauthenticated_metrics_access = true
  }
}
```

**Security Features**:
- **Mutual TLS required** for all client connections
- **Step-CA certificates** with multiple Subject Alternative Names
- **Automatic certificate renewal** via systemd timers
- **Telemetry access** for monitoring without authentication

## Certificate Management Integration

### Step-CA Integration

Vault certificates are issued by the cluster's Step-CA with comprehensive SAN coverage:

**Certificate Attributes**:
- `vault.service.consul` - Service discovery name
- `localhost` - Local access
- Node hostname (e.g., `bettley.nodes.goldentooth.net`)
- Node IP address (e.g., `10.4.0.11`)

**Renewal Automation**:
```ini
[Service]
Environment=CERT_LOCATION=/opt/vault/tls/tls.crt \
            KEY_LOCATION=/opt/vault/tls/tls.key

# Restart Vault service after certificate renewal
ExecStartPost=/usr/bin/env sh -c "! systemctl --quiet is-active vault.service || systemctl try-reload-or-restart vault.service"
```

### Certificate Lifecycle

- **Validity**: 24 hours (short-lived certificates)
- **Renewal**: Automatic via `cert-renewer@vault.timer`
- **Service Integration**: Automatic Vault restart after renewal
- **CLI Management**: `goldentooth rotate_vault_certs`

## Consul Backend Configuration

### Dedicated ACL Policy

Vault nodes use dedicated Consul ACL tokens with specific permissions:

```hcl
key_prefix "vault/" {
  policy = "write"
}

service "vault" {
  policy = "write"
}

agent_prefix "" {
  policy = "read"
}

session_prefix "" {
  policy = "write"
}
```

This provides:
- **Minimal permissions** for Vault's operational needs
- **Isolated key space** under `vault/` prefix
- **Service registration** capabilities
- **Session management** for locking mechanisms

## Security and Service Configuration

### Systemd Hardening

```ini
[Service]
User=vault
Group=vault
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
NoNewPrivileges=yes
```

**Security Measures**:
- **Dedicated user/group** isolation
- **Capability restrictions** - only IPC_LOCK and SYSLOG
- **Memory locking** capability for sensitive data
- **No privilege escalation** permitted

### Environment Security

AWS credentials for KMS access are managed through environment files:

```bash
AWS_ACCESS_KEY_ID={{ vault.aws.access_key_id }}
AWS_SECRET_ACCESS_KEY={{ vault.aws.secret_access_key }}
AWS_REGION={{ vault.aws.region }}
```

- **File permissions**: 0600 (owner read/write only)
- **Encrypted at rest** in Ansible vault
- **Least privilege** IAM policies for KMS access only

## Monitoring and Observability

### Prometheus Integration

```hcl
telemetry {
  prometheus_retention_time = "24h"
  usage_gauge_period = "10m"
  maximum_gauge_cardinality = 500
  enable_hostname_label = true
  lease_metrics_epsilon = "1h"
  num_lease_metrics_buckets = 168
  add_lease_metrics_namespace_labels = false
  filter_default = true
  disable_hostname = true
}
```

**Metrics Features**:
- **24-hour retention** for operational metrics
- **10-minute usage gauges** for capacity planning
- **Hostname labeling** for per-node identification
- **Lease metrics** with weekly granularity (168 buckets)
- **Unauthenticated metrics access** for Prometheus scraping

## Command Line Integration

### goldentooth CLI Commands

```bash
# Deploy and configure Vault cluster
goldentooth setup_vault

# Rotate TLS certificates
goldentooth rotate_vault_certs

# Edit encrypted secrets
goldentooth edit_vault
```

### Environment Configuration

For Vault CLI operations:
```bash
export VAULT_ADDR=https://{{ ipv4_address }}:8200
export VAULT_CLIENT_CERT=/opt/vault/tls/tls.crt
export VAULT_CLIENT_KEY=/opt/vault/tls/tls.key
```

## External Secrets Integration

### Kubernetes Integration

The cluster includes External Secrets Operator (v0.9.13) for Kubernetes secrets management:

- **Namespace**: `external-secrets`
- **Management**: Argo CD GitOps deployment
- **Integration**: Direct Vault API access for secret retrieval
- **Use Cases**: Database credentials, API keys, TLS certificates

## Directory Structure

```
/opt/vault/                 # Base directory
├── tls/                   # TLS certificates
│   ├── tls.crt           # Server certificate (Step-CA issued)
│   └── tls.key           # Private key
├── data/                 # Data directory (unused with Consul backend)
└── raft/                 # Raft storage (unused with Consul backend)

/etc/vault.d/              # Configuration directory
├── vault.hcl             # Main configuration
└── vault.env             # Environment variables (AWS credentials)
```

## High Availability and Operations

### Cluster Behavior

- **Leader Election**: Automatic through Consul backend
- **Split-Brain Protection**: Consul quorum requirements
- **Rolling Updates**: One node at a time with certificate renewal
- **Disaster Recovery**: AWS KMS auto-unsealing enables rapid recovery

### Operational Patterns

**Health Checks**: Consul health checks monitor Vault API availability
**Service Discovery**: `vault.service.consul` provides load balancing
**Monitoring**: Prometheus metrics for capacity and performance monitoring
**Logging**: systemd journal integration with structured logging

That said, I haven't actually put anything into it yet, so the real test will come when I start using it for secrets management across the cluster infrastructure. The External Secrets integration provides the foundation for Kubernetes secrets management, while the Consul integration enables broader service authentication.
