# Dynamic Service Discovery with Consul + HAProxy Dataplane API

Building upon our [HAProxy Dataplane API integration](./067_haproxy_dataplane_api.md), the next architectural evolution was implementing dynamic service discovery. This transformation moved the cluster away from static backend configurations toward a fully dynamic, Consul-driven service mesh architecture where services can relocate between nodes without manual load balancer reconfiguration.

## The Static Configuration Problem

Traditional HAProxy configurations require explicit backend server definitions:

```haproxy
backend grafana-backend
    server grafana1 10.4.1.15:3000 check ssl verify none
    server grafana2 10.4.1.16:3000 check ssl verify none backup
```

This approach creates several operational challenges:

- **Manual Updates**: Adding or removing services requires HAProxy configuration changes
- **Node Dependencies**: Services tied to specific IP addresses can't migrate freely
- **Health Check Duplication**: Both HAProxy and service discovery systems monitor health
- **Configuration Drift**: Static configurations become outdated as infrastructure evolves

## Dynamic Service Discovery Architecture

The new implementation leverages Consul's service registry with HAProxy Dataplane API's dynamic backend creation:

```
Service Registration → Consul Service Registry → HAProxy Dataplane API → Dynamic Backends
```

### Core Components

1. **Consul Service Registry**: Central service discovery database
2. **Service Registration Template**: Reusable Ansible template for consistent service registration
3. **HAProxy Dataplane API**: Dynamic backend management interface
4. **Service-to-Backend Mappings**: Configuration linking Consul services to HAProxy backends

## Implementation: Reusable Service Registration Template

The foundation of dynamic service discovery is the `consul-service-registration.json.j2` template in the `goldentooth.setup_consul` role:

```json
{
  "name": "{{ consul_service_name }}",
  "id": "{{ consul_service_name }}-{{ ansible_hostname }}",
  "address": "{{ consul_service_address | default(ipv4_address) }}",
  "port": {{ consul_service_port }},
  "tags": {{ consul_service_tags | default(['goldentooth']) | to_json }},
  "meta": {
    "version": "{{ consul_service_version | default('unknown') }}",
    "environment": "{{ consul_service_environment | default('production') }}",
    "service_type": "{{ consul_service_type | default('application') }}",
    "cluster": "goldentooth",
    "hostname": "{{ ansible_hostname }}",
    "protocol": "{{ consul_service_protocol | default('http') }}",
    "path": "{{ consul_service_health_path | default('/') }}"
  },
  "checks": [
    {
      "id": "{{ consul_service_name }}-http-health",
      "name": "{{ consul_service_name | title }} HTTP Health Check",
      "http": "{{ consul_service_health_http }}",
      "method": "{{ consul_service_health_method | default('GET') }}",
      "interval": "{{ consul_service_health_interval | default('30s') }}",
      "timeout": "{{ consul_service_health_timeout | default('10s') }}",
      "status": "passing"
    }
  ]
}
```

This template provides:
- **Standardized Service Registration**: Consistent metadata and health check patterns
- **Flexible Health Checks**: HTTP and TCP checks with configurable endpoints
- **Rich Metadata**: Protocol, version, and environment information for routing decisions
- **Health Check Integration**: Native Consul health monitoring replacing static HAProxy checks

## Service Integration Patterns

### Grafana Service Registration

The `goldentooth.setup_grafana` role demonstrates the integration pattern:

```yaml
- name: Register Grafana with Consul
  include_role:
    name: goldentooth.setup_consul
    tasks_from: register_service
  vars:
    consul_service_name: grafana
    consul_service_port: 3000
    consul_service_tags:
      - monitoring
      - dashboard
      - goldentooth
      - https
    consul_service_type: monitoring
    consul_service_protocol: https
    consul_service_health_path: /api/health
    consul_service_health_http: "https://{{ ipv4_address }}:3000/api/health"
    consul_service_health_tls_skip_verify: true
```

This registration creates a Grafana service entry in Consul with:
- **HTTPS Health Checks**: Direct validation of Grafana's API endpoint
- **Service Metadata**: Rich tagging for service discovery and routing
- **TLS Configuration**: Proper SSL handling for encrypted services

### Service-Specific Health Check Endpoints

Each service uses appropriate health check endpoints:

- **Grafana**: `/api/health` - Grafana's built-in health endpoint  
- **Prometheus**: `/-/healthy` - Standard Prometheus health check
- **Loki**: `/ready` - Loki readiness endpoint
- **MCP Server**: `/health` - Custom health endpoint

## HAProxy Dataplane API Configuration

The `dataplaneapi.yaml.j2` template defines service-to-backend mappings:

```yaml
service_discovery:
  consuls:
    - address: 127.0.0.1:8500
      enabled: true
      services:
        - name: grafana
          backend_name: consul-grafana
          mode: http
          balance: roundrobin
          check: enabled
          check_ssl: enabled
          check_path: /api/health
          ssl: enabled
          ssl_verify: none
          
        - name: prometheus  
          backend_name: consul-prometheus
          mode: http
          balance: roundrobin
          check: enabled
          check_path: /-/healthy
          
        - name: loki
          backend_name: consul-loki
          mode: http
          balance: roundrobin
          check: enabled
          check_ssl: enabled
          check_path: /ready
          ssl: enabled
          ssl_verify: none
```

This configuration:
- **Maps Consul Services**: Links service registry entries to HAProxy backends
- **Configures SSL Settings**: Handles HTTPS services with appropriate SSL verification
- **Defines Load Balancing**: Sets algorithm and health check behavior per service
- **Creates Dynamic Backends**: Automatically generates `consul-*` backend names

## Frontend Routing Transformation

HAProxy frontend configuration transitioned from static to dynamic backends:

### Before: Static Backend References
```haproxy
frontend goldentooth-services
  use_backend grafana-backend if { hdr(host) -i grafana.services.goldentooth.net }
  use_backend prometheus-backend if { hdr(host) -i prometheus.services.goldentooth.net }
```

### After: Dynamic Backend References  
```haproxy
frontend goldentooth-services
  use_backend consul-grafana if { hdr(host) -i grafana.services.goldentooth.net }
  use_backend consul-prometheus if { hdr(host) -i prometheus.services.goldentooth.net }
  use_backend consul-loki if { hdr(host) -i loki.services.goldentooth.net }
  use_backend consul-mcp-server if { hdr(host) -i mcp.services.goldentooth.net }
```

The `consul-*` naming convention distinguishes dynamically managed backends from static ones.

## Multi-Service Role Implementation

Each service role now includes Consul registration:

### Prometheus Registration
```yaml
- name: Register Prometheus with Consul
  include_role:
    name: goldentooth.setup_consul
    tasks_from: register_service
  vars:
    consul_service_name: prometheus
    consul_service_port: 9090
    consul_service_health_http: "http://{{ ipv4_address }}:9090/-/healthy"
```

### Loki Registration
```yaml
- name: Register Loki with Consul
  include_role:
    name: goldentooth.setup_consul
    tasks_from: register_service
  vars:
    consul_service_name: loki
    consul_service_port: 3100
    consul_service_health_http: "https://{{ ipv4_address }}:3100/ready"
    consul_service_health_tls_skip_verify: true
```

### MCP Server Registration
```yaml
- name: Register MCP Server with Consul
  include_role:
    name: goldentooth.setup_consul
    tasks_from: register_service
  vars:
    consul_service_name: mcp-server
    consul_service_port: 3001
    consul_service_health_http: "http://{{ ipv4_address }}:3001/health"
```

## Technical Benefits

### Service Mobility
Services can now migrate between nodes without load balancer reconfiguration. When a service starts on a different node, it registers with Consul, and HAProxy automatically updates backend server lists.

### Health Check Integration
Consul's health checking replaces static HAProxy health checks, providing:
- **Centralized Health Monitoring**: Single source of truth for service health
- **Rich Health Check Types**: HTTP, TCP, script-based, and TTL checks
- **Health Check Inheritance**: HAProxy backends inherit health status from Consul

### Configuration Simplification
Static backend definitions are eliminated, reducing HAProxy configuration complexity and maintenance overhead.

### Service Discovery Foundation
The implementation establishes patterns for:
- **Service Registration**: Standardized across all cluster services
- **Health Check Consistency**: Uniform health monitoring approaches
- **Metadata Management**: Rich service information for advanced routing
- **Dynamic Backend Naming**: Clear separation between static and dynamic backends

## Operational Impact

### Deployment Flexibility
Services can be deployed to any cluster node without infrastructure configuration changes. The service registers itself with Consul, and HAProxy automatically includes it in load balancing.

### Zero-Downtime Updates
Service updates can leverage Consul's health checking for gradual rollouts. Unhealthy instances are automatically removed from load balancing until they pass health checks.

### Monitoring Integration  
Consul's web UI provides real-time service health visualization, complementing existing Prometheus/Grafana monitoring infrastructure.

## Future Service Mesh Evolution

This implementation represents the foundation for comprehensive service mesh architecture:

- **Additional Service Registration**: Extending dynamic discovery to all cluster services
- **Advanced Routing**: Consul metadata-based traffic routing and service versioning
- **Security Integration**: Service-to-service authentication and authorization
- **Circuit Breaking**: Automated failure handling and traffic management

The transformation from static to dynamic service discovery fundamentally changes how the Goldentooth cluster manages service routing, establishing patterns that will support continued infrastructure evolution and automation.