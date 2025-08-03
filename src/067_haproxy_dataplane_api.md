# HAProxy Dataplane API Integration

With the cluster's load balancing infrastructure established through [our initial HAProxy setup](./006_load_balancer.md) and [subsequent revisiting](./035_load_balancer_revisited.md), the next evolution was to enable dynamic configuration management. HAProxy's traditional configuration model requires service restarts for changes, which creates service disruption and doesn't align well with modern infrastructure automation needs.

The HAProxy Dataplane API provides a RESTful interface for runtime configuration management, allowing backend server manipulation, health check configuration, and statistics collection without HAProxy restarts. This capability is essential for automated deployment pipelines and dynamic infrastructure management.

## Implementation Strategy

The implementation focused on integrating HAProxy Dataplane API v3.2.1 into the existing `goldentooth.setup_haproxy` Ansible role while maintaining the cluster's security and operational standards.

### Configuration Architecture

The API requires a specific YAML v2 configuration format with a nested structure significantly different from HAProxy's traditional flat configuration:

```yaml
config_version: 2
haproxy:
  config_file: /etc/haproxy/haproxy.cfg
  userlist: controller
  reload:
    reload_cmd: systemctl reload haproxy
    reload_delay: 5
    restart_cmd: systemctl restart haproxy
name: dataplaneapi
mode: single
resources:
  maps_dir: /etc/haproxy/maps
  ssl_certs_dir: /etc/haproxy/ssl
  general_storage_dir: /etc/haproxy/general
  spoe_dir: /etc/haproxy/spoe
  spoe_transaction_dir: /tmp/spoe-haproxy
  backups_dir: /etc/haproxy/backups
  config_snippets_dir: /etc/haproxy/config_snippets
  acl_dir: /etc/haproxy/acl
  transactions_dir: /etc/haproxy/transactions
user:
  insecure: false
  username: "{{ vault.cluster_credentials.username }}"
  password: "{{ vault.cluster_credentials.password }}"
advertised:
  api_address: 0.0.0.0
  api_port: 5555
```

This configuration structure enables the API to manage HAProxy through systemd reload commands rather than requiring full restarts, maintaining service availability during configuration changes.

### Directory Structure Implementation

The API requires an extensive directory hierarchy for storing various configuration components:

```bash
# Primary API configuration
/etc/haproxy-dataplane/

# HAProxy configuration storage
/etc/haproxy/dataplane/
/etc/haproxy/maps/
/etc/haproxy/ssl/
/etc/haproxy/general/
/etc/haproxy/spoe/
/etc/haproxy/acl/
/etc/haproxy/transactions/
/etc/haproxy/config_snippets/
/etc/haproxy/backups/

# Temporary processing
/tmp/spoe-haproxy/
```

All directories are created with proper ownership (`haproxy:haproxy`) and permissions to ensure the API service can read and write configuration data while maintaining security boundaries.

## HAProxy Configuration Integration

The implementation required specific HAProxy configuration changes to enable API communication:

### Master-Worker Mode

```haproxy
global
    master-worker
    
    # Admin socket with proper group permissions
    stats socket /run/haproxy/admin.sock mode 660 level admin group haproxy
    
    # User authentication for API access
    userlist controller
        user {{ vault.cluster_credentials.username }} password {{ vault.cluster_credentials.password }}
```

The master-worker mode enables the API to communicate with HAProxy's runtime through the admin socket, while the userlist provides authentication for API requests.

### Backend Configuration

```haproxy
backend haproxy-dataplane-api
    server dataplane 127.0.0.1:5555 check
```

This backend configuration allows external access to the API through the existing reverse proxy infrastructure, integrating seamlessly with the cluster's routing patterns.

## Systemd Service Implementation

The service configuration prioritizes security while providing necessary filesystem access:

```systemd
[Unit]
Description=HAProxy Dataplane API
After=network.target haproxy.service
Requires=haproxy.service

[Service]
Type=exec
User=haproxy
Group=haproxy
ExecStart=/usr/local/bin/dataplaneapi --config-file=/etc/haproxy-dataplane/dataplaneapi.yaml
Restart=always
RestartSec=5

# Security settings
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true

# Required filesystem access
ReadWritePaths=/etc/haproxy
ReadWritePaths=/etc/haproxy-dataplane
ReadWritePaths=/var/lib/haproxy
ReadWritePaths=/run/haproxy
ReadWritePaths=/tmp/spoe-haproxy

[Install]
WantedBy=multi-user.target
```

The security-focused configuration uses `ProtectSystem=strict` with explicit `ReadWritePaths` declarations, ensuring the service has access only to required directories while maintaining system protection.

## Problem Resolution Process

The implementation encountered several configuration challenges that required systematic debugging:

### YAML Configuration Format Issues

**Problem**: Initial configuration used HAProxy's flat format rather than the required nested YAML v2 structure.

**Solution**: Implemented proper `config_version: 2` with nested `haproxy:` sections and structured resource directories.

### Socket Permission Problems

**Problem**: HAProxy admin socket was inaccessible to the dataplane API service.

```
ERRO[0000] error fetching configuration: dial unix /run/haproxy/admin.sock: connect: permission denied
```

**Solution**: Added `group haproxy` to the HAProxy socket configuration, allowing the dataplane API service running as the haproxy user to access the socket.

### Directory Permission Resolution

**Problem**: Multiple permission denied errors for various storage directories.

```
ERRO[0000] Cannot create dir /etc/haproxy/maps: mkdir /etc/haproxy/maps: permission denied
```

**Solution**: Systematically created all required directories with proper ownership:

```ansible
- name: Create HAProxy dataplane directories
  file:
    path: "{{ item }}"
    state: directory
    owner: haproxy
    group: haproxy
    mode: '0755'
  loop:
    - /etc/haproxy/dataplane
    - /etc/haproxy/maps
    - /etc/haproxy/ssl
    - /etc/haproxy/general
    - /etc/haproxy/spoe
    - /etc/haproxy/acl
    - /etc/haproxy/transactions
    - /etc/haproxy/config_snippets
    - /etc/haproxy/backups
    - /tmp/spoe-haproxy
```

### Filesystem Write Access

**Problem**: The `/etc/haproxy` directory was read-only for the haproxy user, preventing configuration updates.

**Solution**: Modified directory ownership and permissions to allow write access while maintaining security:

```bash
chgrp haproxy /etc/haproxy
chmod g+w /etc/haproxy
```

## Service Integration and Accessibility

The API integrates with the cluster's existing infrastructure patterns:

- **Service Discovery**: Available at `https://haproxy-api.services.goldentooth.net`
- **Authentication**: Uses cluster credentials for API access
- **Monitoring**: Integrated with existing health check patterns
- **Security**: TLS termination through existing certificate management

## Operational Capabilities

The successful implementation enables several advanced load balancer management capabilities:

### Dynamic Backend Management

```bash
# Add backend servers without HAProxy restart
curl -X POST https://haproxy-api.services.goldentooth.net/v3/services/haproxy/configuration/servers \
  -d '{"name": "new-server", "address": "10.4.1.10", "port": 8080}'

# Modify server weights for traffic distribution
curl -X PUT https://haproxy-api.services.goldentooth.net/v3/services/haproxy/configuration/servers/web1 \
  -d '{"weight": 150}'
```

### Health Check Configuration

```bash
# Configure health checks dynamically
curl -X PUT https://haproxy-api.services.goldentooth.net/v3/services/haproxy/configuration/backends/web \
  -d '{"health_check": {"uri": "/health", "interval": "5s"}}'
```

### Runtime Statistics and Monitoring

The API provides comprehensive runtime statistics and configuration state information, enabling advanced monitoring and automated decision-making for infrastructure management.

## Current Status and Integration

The HAProxy Dataplane API is now:

- **Active and stable** on the allyrion load balancer node
- **Listening on port 5555** with proper systemd management
- **Responding to HTTP API requests** with full functionality
- **Integrated with HAProxy** through the admin socket interface
- **Accessible externally** via the configured domain endpoint
- **Authenticated** using cluster credential standards

This implementation represents a significant enhancement to the cluster's load balancing capabilities, moving from static configuration management to dynamic, API-driven infrastructure control. The systematic approach to troubleshooting configuration issues demonstrates the methodical problem-solving required for complex infrastructure integration while maintaining operational security and reliability standards.