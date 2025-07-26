# Authelia Authentication Infrastructure

In our quest to provide secure access to the Goldentooth cluster for AI assistants, we needed a robust authentication and authorization solution. This chapter chronicles the implementation of Authelia, a comprehensive authentication server that provides OAuth 2.0 and OpenID Connect capabilities for our cluster services.

## The Authentication Challenge

As we began developing the MCP (Model Context Protocol) server to enable AI assistants like Claude Code to interact with our cluster, we faced a critical security requirement: how to provide secure, standards-based authentication without compromising cluster security or creating a poor user experience.

Traditional authentication approaches like API keys or basic authentication felt inadequate for this use case. We needed:

- Standards-based OAuth 2.0 and OpenID Connect support
- Multi-factor authentication capabilities
- Fine-grained authorization policies
- Integration with our existing Step-CA certificate infrastructure
- Single Sign-On (SSO) for multiple cluster services

## Why Authelia?

After evaluating various authentication solutions, Authelia emerged as the ideal choice for our cluster:

- **Comprehensive Feature Set**: OAuth 2.0, OpenID Connect, LDAP, 2FA/MFA support
- **Self-Hosted**: No dependency on external authentication providers
- **Lightweight**: Perfect for deployment on Raspberry Pi infrastructure
- **Flexible Storage**: Supports SQLite for simplicity or PostgreSQL for scale
- **Policy Engine**: Fine-grained access control based on users, groups, and resources

## Architecture Overview

Authelia fits into our cluster architecture as the central authentication authority:

```
Claude Code (OAuth Client)
    ↓ OAuth 2.0 Authorization Code Flow
Authelia (auth.services.goldentooth.net)
    ↓ JWT/Token Validation
MCP Server (mcp.services.goldentooth.net)
    ↓ Authenticated API Calls
Goldentooth Cluster Services
```

The authentication flow follows industry-standard OAuth 2.0 patterns:

1. **Discovery**: Client discovers OAuth endpoints via well-known URLs
2. **Authorization**: User authenticates with Authelia and grants permissions
3. **Token Exchange**: Authorization code exchanged for access/ID tokens
4. **API Access**: Bearer tokens used for authenticated MCP requests

## Ansible Implementation

### Role Structure

The `goldentooth.setup_authelia` role provides comprehensive deployment automation:

```
ansible/roles/goldentooth.setup_authelia/
├── defaults/main.yml      # Default configuration variables
├── tasks/main.yml         # Primary deployment tasks
├── templates/             # Configuration templates
│   ├── configuration.yml.j2           # Main Authelia config
│   ├── users_database.yml.j2          # User definitions
│   ├── authelia.service.j2             # Systemd service
│   ├── authelia-consul-service.json.j2 # Consul registration
│   └── cert-renewer@authelia.conf.j2   # Certificate renewal
├── handlers/main.yml      # Service restart handlers
└── CLAUDE.md             # Role documentation
```

### Key Configuration Elements

**OIDC Provider Configuration**: Authelia acts as a full OpenID Connect provider with pre-configured clients for the MCP server:

```yaml
identity_providers:
  oidc:
    hmac_secret: {{ authelia_oidc_hmac_secret }}
    clients:
      - client_id: goldentooth-mcp
        client_name: Goldentooth MCP Server
        client_secret: "$argon2id$v=19$m=65536,t=3,p=4$..."
        authorization_policy: one_factor
        redirect_uris:
          - https://mcp.services.{{ authelia_domain }}/callback
        scopes:
          - openid
          - profile
          - email
          - groups
          - offline_access
        grant_types:
          - authorization_code
          - refresh_token
```

**Security Hardening**: Multiple layers of security protection:

```yaml
authentication_backend:
  file:
    password:
      algorithm: argon2id
      iterations: 3
      memory: 65536
      parallelism: 4
      key_length: 32
      salt_length: 16

regulation:
  max_retries: 3
  find_time: 2m
  ban_time: 5m

session:
  secret: {{ authelia_session_secret }}
  expiration: 12h
  inactivity: 45m
```

### Certificate Integration

Authelia integrates seamlessly with our Step-CA infrastructure:

```bash
# Generate TLS certificate for Authelia server
step ca certificate \
  "authelia.{{ authelia_domain }}" \
  /etc/authelia/tls.crt \
  /etc/authelia/tls.key \
  --provisioner="default" \
  --san="authelia.{{ authelia_domain }}" \
  --san="auth.services.{{ authelia_domain }}" \
  --not-after='24h' \
  --force
```

The role also configures automatic certificate renewal through our `cert-renewer@authelia.timer` service, ensuring continuous operation without manual intervention.

## Consul Integration

Authelia registers itself as a service in our Consul service mesh, enabling service discovery and health monitoring:

```json
{
  "service": {
    "name": "authelia",
    "port": 9091,
    "address": "{{ ansible_hostname }}.{{ cluster.node_domain }}",
    "tags": ["authentication", "oauth", "oidc"],
    "check": {
      "http": "https://{{ ansible_hostname }}.{{ cluster.node_domain }}:9091/api/health",
      "interval": "30s",
      "timeout": "10s",
      "tls_skip_verify": false
    }
  }
}
```

This integration provides:
- **Service Discovery**: Other services can locate Authelia via Consul DNS
- **Health Monitoring**: Consul tracks Authelia's health status
- **Load Balancing**: Support for multiple Authelia instances if needed

## User Management and Policies

### Default User Configuration

The deployment creates essential user accounts:

```yaml
users:
  admin:
    displayname: "Administrator"
    password: "$argon2id$v=19$m=65536,t=3,p=4$..."
    email: admin@goldentooth.net
    groups:
      - admins
      - users

  mcp-service:
    displayname: "MCP Service Account"
    password: "$argon2id$v=19$m=65536,t=3,p=4$..."
    email: mcp-service@goldentooth.net
    groups:
      - services
```

### Access Control Policies

Authelia implements fine-grained access control:

```yaml
access_control:
  default_policy: one_factor
  rules:
    # Public access to health checks
    - domain: "*.{{ authelia_domain }}"
      policy: bypass
      resources:
        - "^/api/health$"

    # Admin resources require two-factor
    - domain: "*.{{ authelia_domain }}"
      policy: two_factor
      subject:
        - "group:admins"

    # Regular user access
    - domain: "*.{{ authelia_domain }}"
      policy: one_factor
```

## Multi-Factor Authentication

Authelia supports multiple 2FA methods out of the box:

**TOTP (Time-based One-Time Password)**:
- Compatible with Google Authenticator, Authy, 1Password
- 6-digit codes with 30-second rotation
- QR code enrollment process

**WebAuthn/FIDO2**:
- Hardware security keys (YubiKey, SoloKey)
- Platform authenticators (TouchID, Windows Hello)
- Phishing-resistant authentication

**Push Notifications** (planned):
- Integration with Duo Security for push-based 2FA
- SMS fallback for environments without smartphone access

## Deployment and Management

### Installation Command

Deploy Authelia across the cluster with a single command:

```bash
# Deploy to default Authelia nodes
goldentooth setup_authelia

# Deploy to specific node
goldentooth setup_authelia --limit jast
```

### Service Management

Monitor and manage Authelia using familiar systemd commands:

```bash
# Check service status
goldentooth command authelia "systemctl status authelia"

# View logs
goldentooth command authelia "journalctl -u authelia -f"

# Restart service
goldentooth command_root authelia "systemctl restart authelia"

# Validate configuration
goldentooth command authelia "/usr/local/bin/authelia validate-config --config /etc/authelia/configuration.yml"
```

### Health Monitoring

Authelia exposes health and metrics endpoints:

- **Health Check**: `https://auth.goldentooth.net/api/health`
- **Metrics**: `http://auth.goldentooth.net:9959/metrics` (Prometheus format)

These endpoints integrate with our monitoring stack (Prometheus, Grafana) for observability.

## Security Considerations

### Threat Mitigation

Authelia addresses multiple attack vectors:

**Session Security**:
- Secure, HTTP-only cookies
- CSRF protection via state parameters
- Session timeout and inactivity limits

**Rate Limiting**:
- Failed login attempt throttling
- IP-based temporary bans
- Progressive delays for repeated failures

**Password Security**:
- Argon2id hashing (memory-hard, side-channel resistant)
- Configurable complexity requirements
- Protection against timing attacks

### Network Security

All Authelia communication is secured:

- **TLS 1.3**: All external communications encrypted
- **Certificate Validation**: Mutual TLS with cluster CA
- **HSTS**: HTTP Strict Transport Security headers
- **Secure Headers**: Complete security header suite

## Integration with MCP Server

The MCP server integrates with Authelia through standard OAuth 2.0 flows:

### OAuth Discovery

The MCP server exposes OAuth discovery endpoints that delegate to Authelia:

```rust
// In http_server.rs
async fn handle_oauth_metadata() -> Result<Response<Full<Bytes>>, Infallible> {
    let discovery = auth.discover_oidc_config().await?;
    let metadata = serde_json::json!({
        "issuer": discovery.issuer,
        "authorization_endpoint": discovery.authorization_endpoint,
        "token_endpoint": discovery.token_endpoint,
        "jwks_uri": discovery.jwks_uri,
        // ... additional OAuth metadata
    });
    Ok(Response::builder()
        .status(StatusCode::OK)
        .header("Content-Type", "application/json")
        .body(Full::new(Bytes::from(metadata.to_string())))
        .unwrap())
}
```

### Token Validation

The MCP server validates tokens using both JWT verification and OAuth token introspection:

```rust
async fn validate_token(&self, token: &str) -> AuthResult<Claims> {
    if self.is_jwt_token(token) {
        // JWT validation for ID tokens
        self.validate_jwt_token(token).await
    } else {
        // Token introspection for opaque access tokens
        self.introspect_access_token(token).await
    }
}
```

This dual approach supports both JWT ID tokens and opaque access tokens that Authelia issues.

## Performance and Scalability

### Resource Utilization

Authelia runs efficiently on Raspberry Pi hardware:

- **Memory**: ~50MB RSS under normal load
- **CPU**: <1% utilization during authentication flows
- **Storage**: SQLite database grows slowly (~10MB for hundreds of users)
- **Network**: Minimal bandwidth requirements

### Scaling Strategies

For high-availability deployments:

1. **Multiple Instances**: Deploy Authelia on multiple nodes with shared database
2. **PostgreSQL Backend**: Replace SQLite with PostgreSQL for concurrent access
3. **Redis Sessions**: Use Redis for distributed session storage
4. **Load Balancing**: HAProxy or similar for request distribution
