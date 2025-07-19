# Loki

_This, the previous "article" (on [Grafana](./043_grafana.md)), and the next one (on [Vector](./045_vector.md)), are occurring mostly in parallel so that I can validate these services as I go._

Loki is... there's a whole lot going on there.

## Log Retention Configuration

I enabled a retention policy so that my logs wouldn't grow without bound until the end of time. This coincided with me noticing that my `/var/log/journal` directories had gotten up to about 4GB, which led me to perform a similar change in the `journald` configuration.

**Retention Policy Configuration**:
```yaml
limits_config:
  retention_period: 168h  # 7 days

compactor:
  working_directory: /tmp/retention
  compaction_interval: 10m
  retention_enabled: true
  retention_delete_delay: 2h
  retention_delete_worker_count: 5
  delete_request_store: filesystem
```

I reduced the `retention_delete_worker_count` from 150 to 5 🙂 This optimization:
- **Reduces resource usage**: Less CPU overhead on Raspberry Pi nodes
- **Maintains efficiency**: 5 workers sufficient for 7-day retention window
- **Prevents overload**: Avoids overwhelming the Pi's limited resources

## Consul Integration for Ring Management

I also configured Loki to use Consul as its ring kvstore, which involved sketching out an ACL policy and generating a token, but nothing too weird. (Assuming that it works.)

**Ring Configuration**:
```yaml
common:
  ring:
    kvstore:
      store: consul
      consul:
        acl_token: {{ loki_consul_token }}
        host: {{ ipv4_address }}:8500
```

**Consul ACL Policy** (`loki.policy.hcl`):
```hcl
key_prefix "collectors/" {
  policy = "write"
}

key_prefix "loki/" {
  policy = "write"
}
```

This integration provides:
- **Service discovery**: Automatic discovery of Loki components
- **Consistent hashing**: Proper ring distribution for ingester scaling
- **High availability**: Shared state management across cluster nodes
- **Security**: ACL-based access control to Consul KV store

## Comprehensive TLS Configuration

The next several hours involved cleanup after I rashly configured Loki to use TLS. I didn't know that I'd then need to configure Loki to communicate with itself via TLS, and that I would have to do so in several different places and that those places would have different syntax for declaring the same core ideas (CA cert, TLS cert, TLS key).

### Server TLS Configuration

**GRPC and HTTP Server**:
```yaml
server:
  grpc_listen_address: {{ ipv4_address }}
  grpc_listen_port: 9096
  grpc_tls_config: &http_tls_config
    cert_file: "{{ loki.cert_path }}"
    key_file: "{{ loki.key_path }}"
    client_ca_file: "{{ step_ca.root_cert_path }}"
    client_auth_type: "VerifyClientCertIfGiven"
  http_listen_address: {{ ipv4_address }}
  http_listen_port: 3100
  http_tls_config: *http_tls_config
```

**TLS Features**:
- **Mutual TLS**: Client certificate verification when provided
- **Step-CA Integration**: Uses cluster certificate authority
- **YAML Anchors**: Reuses TLS config across components to reduce duplication

### Component-Level TLS Configuration

**Frontend Configuration**:
```yaml
frontend:
  grpc_client_config: &grpc_client_config
    tls_enabled: true
    tls_cert_path: "{{ loki.cert_path }}"
    tls_key_path: "{{ loki.key_path }}"
    tls_ca_path: "{{ step_ca.root_cert_path }}"
  tail_tls_config:
    tls_cert_path: "{{ loki.cert_path }}"
    tls_key_path: "{{ loki.key_path }}"
    tls_ca_path: "{{ step_ca.root_cert_path }}"
```

**Pattern Ingester TLS**:
```yaml
pattern_ingester:
  metric_aggregation:
    loki_address: {{ ipv4_address }}:3100
    use_tls: true
    http_client_config:
      tls_config:
        ca_file: "{{ step_ca.root_cert_path }}"
        cert_file: "{{ loki.cert_path }}"
        key_file: "{{ loki.key_path }}"
```

### Internal Component Communication

The configuration ensures TLS across all internal communications:

- **Ingester Client**: `grpc_client_config: *grpc_client_config`
- **Frontend Worker**: `grpc_client_config: *grpc_client_config`
- **Query Scheduler**: `grpc_client_config: *grpc_client_config`
- **Ruler**: Uses separate alertmanager client TLS config

And holy crap, the Loki site is absolutely awful for finding and understanding where some configuration is needed.

## Advanced Configuration Features

### Pattern Recognition and Analytics

**Pattern Ingester**:
```yaml
pattern_ingester:
  enabled: true
  metric_aggregation:
    loki_address: {{ ipv4_address }}:3100
    use_tls: true
```

This enables:
- **Log pattern detection**: Automatic recognition of log patterns
- **Metric generation**: Convert log patterns to Prometheus metrics
- **Performance insights**: Understanding log volume and patterns

### Schema and Storage Configuration

**TSDB Schema** (v13):
```yaml
schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h
```

**Storage Paths**:
```yaml
common:
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
```

### Query Performance Optimization

**Caching Configuration**:
```yaml
query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 20
```

**Performance Features**:
- **Embedded cache**: 20MB query result cache for faster repeated queries
- **Protobuf encoding**: Efficient data serialization for frontend communication
- **Concurrent streams**: 1000 max concurrent GRPC streams

## Certificate Management Integration

**Automatic Certificate Renewal**:
```ini
[Service]
Environment=CERT_LOCATION={{ loki.cert_path }} \
            KEY_LOCATION={{ loki.key_path }}

# Restart Loki service after certificate renewal
ExecStartPost=/usr/bin/env sh -c "! systemctl --quiet is-active loki.service || systemctl try-reload-or-restart loki.service"
```

**Certificate Lifecycle**:
- **24-hour validity**: Short-lived certificates for enhanced security
- **Automatic renewal**: `cert-renewer@loki.timer` handles renewal
- **Service restart**: Seamless certificate updates with service reload
- **Step-CA integration**: Consistent with cluster-wide PKI infrastructure

## Monitoring and Alerting Integration

**Ruler Configuration**:
```yaml
ruler:
  alertmanager_url: http://{{ ipv4_address }}:9093
  alertmanager_client:
    tls_cert_path: "{{ loki.cert_path }}"
    tls_key_path: "{{ loki.key_path }}"
    tls_ca_path: "{{ step_ca.root_cert_path }}"
```

**Observability Features**:
- **Structured logging**: JSON format for better parsing
- **Debug logging**: Detailed logging for troubleshooting
- **Request logging**: Log requests at info level for monitoring
- **Grafana integration**: Primary storage for alert state history

## Deployment Architecture

**Single-Node Deployment**: Currently deployed on `inchfield` node
**Replication Factor**: 1 (appropriate for single-node setup)
**Resource Optimization**: Configured for Raspberry Pi resource constraints
**Integration Points**: 
- **Vector**: Log shipping from all cluster nodes
- **Grafana**: Log visualization and alerting
- **Prometheus**: Metrics scraping from Loki endpoints

This comprehensive Loki configuration provides a production-ready log aggregation platform with enterprise-grade security, retention management, and integration capabilities, despite the complexity of getting all the TLS configurations properly aligned across the numerous internal components.
