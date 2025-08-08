# MCP Server Implementation: Bridging AI and Infrastructure

With our Authelia authentication infrastructure in place, we could finally tackle one of the most ambitious goals of the Goldentooth project: creating a bridge between AI assistants and our cluster infrastructure. This chapter details the implementation of our MCP (Model Context Protocol) server, a Rust-based service that enables AI assistants like Claude Code to securely interact with cluster management tools.

## The Vision: AI-Powered Infrastructure Management

The idea emerged from a simple observation: modern AI assistants excel at understanding complex systems and executing multi-step procedures, but they lack direct access to infrastructure tools. What if we could provide secure, controlled access to cluster management commands through a standardized protocol?

This led us to the Model Context Protocol (MCP), an open standard developed by Anthropic for connecting AI assistants to external tools and data sources. By implementing an MCP server, we could:

- Provide AI assistants with real-time cluster status information
- Enable automated troubleshooting and maintenance tasks
- Create a natural language interface for complex cluster operations
- Maintain strict security boundaries through OAuth 2.0 authentication

## MCP Protocol Overview

The Model Context Protocol operates on a client-server architecture using JSON-RPC 2.0:

```
AI Assistant (Claude Code) ←→ MCP Client ←→ MCP Server ←→ Cluster Tools
```

Key protocol features:
- **Tools**: Discrete functions the AI can invoke (e.g., "check cluster status")
- **Resources**: Data sources the AI can read (e.g., log files, configurations)
- **Prompts**: Reusable templates for common operations
- **Sampling**: AI-assisted content generation

Our implementation focuses primarily on tools, providing a comprehensive set of cluster management functions.

## Architecture Overview

### Core Components

The MCP server is built in Rust using modern async patterns:

```rust
// Core service implementing MCP protocol
pub struct GoldentoothService {
    // MCP service implementation
}

// HTTP server wrapper for web-based access
pub struct HttpServer {
    service: GoldentoothService,
    auth_service: Option<AuthService>,
}

// OAuth 2.0 authentication integration
pub struct AuthService {
    config: AuthConfig,
    client: Client,
    oauth_client: Option<BasicClient>,
    // ... additional fields
}
```

### Dual Transport Support

The server supports both transport modes required by different use cases:

**1. stdin/stdout Mode** (Traditional MCP):
```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let service = GoldentoothService::new();
    let transport = (stdin(), stdout());
    let server = service.serve(transport).await?;
    let _quit_reason = server.waiting().await?;
    Ok(())
}
```

**2. HTTP Mode** (Web-based with OAuth):
```rust
// HTTP server mode with authentication
if env::var("MCP_HTTP_MODE").is_ok() || env::args().any(|arg| arg == "--http") {
    let port = env::var("MCP_PORT").unwrap_or_else(|_| "8080".to_string());
    let addr = format!("0.0.0.0:{}", port).parse()?;

    let http_server = HttpServer::new(service, auth_service);
    http_server.serve(addr).await?;
}
```

## Tool Implementation

### Available Tools

The MCP server exposes five core cluster management tools:

**1. cluster_ping**: Basic connectivity check
```json
{
  "name": "cluster_ping",
  "description": "Ping all nodes in the goldentooth cluster to check their status",
  "inputSchema": {
    "type": "object",
    "properties": {}
  }
}
```

**2. cluster_status**: Detailed node information
```json
{
  "name": "cluster_status",
  "description": "Get detailed status information for cluster nodes",
  "inputSchema": {
    "type": "object",
    "properties": {
      "node": {
        "type": "string",
        "description": "Specific node to check (e.g., 'allyrion', 'jast'). If not provided, checks all nodes."
      }
    }
  }
}
```

**3. service_status**: SystemD service monitoring
```json
{
  "name": "service_status",
  "description": "Check the status of systemd services on cluster nodes",
  "inputSchema": {
    "type": "object",
    "properties": {
      "service": {
        "type": "string",
        "description": "Service name to check (e.g., 'consul', 'nomad', 'vault')"
      },
      "node": {
        "type": "string",
        "description": "Specific node to check. If not provided, checks all nodes."
      }
    },
    "required": ["service"]
  }
}
```

**4. resource_usage**: System resource monitoring
```json
{
  "name": "resource_usage",
  "description": "Get memory and disk usage information for cluster nodes",
  "inputSchema": {
    "type": "object",
    "properties": {
      "node": {
        "type": "string",
        "description": "Specific node to check. If not provided, checks all nodes."
      }
    }
  }
}
```

**5. cluster_info**: Comprehensive cluster overview
```json
{
  "name": "cluster_info",
  "description": "Get comprehensive cluster information including node status and service membership",
  "inputSchema": {
    "type": "object",
    "properties": {}
  }
}
```

### Tool Name Prefixing

A recent enhancement added automatic tool name prefixing for integration with external MCP clients:

```rust
// Tools are automatically prefixed when served via MCP
"mcp__goldentooth_mcp__cluster_ping"
"mcp__goldentooth_mcp__cluster_status"
"mcp__goldentooth_mcp__service_status"
"mcp__goldentooth_mcp__resource_usage"
"mcp__goldentooth_mcp__cluster_info"
```

This prevents naming conflicts when multiple MCP servers are connected to the same client.

## HTTP Server Implementation

### Request Routing

The HTTP server implements comprehensive request routing:

```rust
pub async fn handle_request(
    req: Request<hyper::body::Incoming>,
    service: GoldentoothService,
    auth_service: Option<AuthService>,
) -> Result<Response<Full<Bytes>>, Infallible> {

    // Handle CORS preflight
    if req.method() == Method::OPTIONS {
        return handle_cors_preflight();
    }

    // Handle health check endpoint
    if req.method() == Method::GET && req.uri().path() == "/health" {
        return handle_health_check();
    }

    // Handle OAuth well-known endpoints
    if req.uri().path() == "/.well-known/oauth-authorization-server" ||
       req.uri().path() == "/.well-known/openid-configuration" {
        return handle_oauth_metadata(auth_service, req.method()).await;
    }

    // Handle authentication endpoints
    if req.uri().path().starts_with("/auth/") {
        return handle_auth_request(req, auth_service).await;
    }

    // Handle OAuth callback
    if req.method() == Method::GET && req.uri().path() == "/callback" {
        return handle_oauth_callback(&req);
    }

    // Handle MCP JSON-RPC requests
    if req.method() == Method::POST && req.uri().path() == "/mcp/request" {
        // Continue with authenticated MCP request handling...
    }

    // Return 404 for unknown paths
    Ok(Response::builder()
        .status(StatusCode::NOT_FOUND)
        .body(Full::new(Bytes::from("{\"error\":\"Not Found\"}")))
        .unwrap())
}
```

### Security Hardening

The HTTP server implements multiple security measures:

**1. Request Size Limits** (DoS Protection):
```rust
const MAX_REQUEST_BODY_SIZE: usize = 1024 * 1024; // 1MB

async fn collect_request_body_with_size_limit(
    req: Request<hyper::body::Incoming>,
) -> Result<(Bytes, HashMap<String, String>), Response<Full<Bytes>>> {
    // Check content-length header
    if let Some(content_length_str) = headers.get("content-length") {
        if let Ok(content_length) = content_length_str.parse::<usize>() {
            if content_length > MAX_REQUEST_BODY_SIZE {
                return Err(Response::builder()
                    .status(StatusCode::PAYLOAD_TOO_LARGE)
                    .body(Full::new(Bytes::from(format!(
                        "Request body too large: {} bytes exceeds maximum {} bytes",
                        content_length, MAX_REQUEST_BODY_SIZE
                    ))))
                    .unwrap());
            }
        }
    }
    // ... body collection with size validation
}
```

**2. HTML Escaping** (XSS Prevention):
```rust
pub fn html_escape(input: &str) -> String {
    input
        .replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
        .replace('\'', "&#x27;")
}
```

**3. CORS Headers**:
```rust
fn handle_cors_preflight() -> Result<Response<Full<Bytes>>, Infallible> {
    Ok(Response::builder()
        .status(StatusCode::OK)
        .header("Access-Control-Allow-Origin", "*")
        .header("Access-Control-Allow-Methods", "GET, POST, OPTIONS")
        .header("Access-Control-Allow-Headers", "Content-Type, Authorization")
        .body(Full::new(Bytes::new()))
        .unwrap())
}
```

## Authentication Integration

### OAuth 2.0 Flow Implementation

The MCP server acts as both an OAuth client and resource server:

**Discovery Endpoints**: Expose OAuth metadata for client discovery
```rust
async fn handle_oauth_metadata(auth_service: Option<AuthService>) -> Result<Response<Full<Bytes>>, Infallible> {
    let discovery = auth.discover_oidc_config().await?;
    let metadata = serde_json::json!({
        "issuer": discovery.issuer,
        "authorization_endpoint": discovery.authorization_endpoint,
        "token_endpoint": discovery.token_endpoint,
        "jwks_uri": discovery.jwks_uri,
        "response_types_supported": discovery.response_types_supported,
        "grant_types_supported": discovery.grant_types_supported,
        "scopes_supported": discovery.scopes_supported,
        "token_endpoint_auth_methods_supported": ["client_secret_post", "client_secret_basic"],
        "code_challenge_methods_supported": ["S256", "plain"]
    });
    // Return metadata response...
}
```

**Token Exchange**: Proxy authorization code exchange to Authelia
```rust
async fn handle_auth_request(req: Request<hyper::body::Incoming>, auth_service: Option<AuthService>) -> Result<Response<Full<Bytes>>, Infallible> {
    match (req.method(), req.uri().path()) {
        (&Method::POST, "/auth/token") => {
            let body = req.collect().await?.to_bytes();
            let request_data: serde_json::Value = serde_json::from_slice(&body)?;
            let code = request_data.get("code").and_then(|c| c.as_str())?;

            match auth.exchange_code_for_token(code).await {
                Ok(access_token) => {
                    let response_data = serde_json::json!({
                        "access_token": access_token.secret(),
                        "token_type": "Bearer",
                        "expires_in": 3600
                    });
                    // Return successful token response...
                }
                Err(e) => {
                    // Handle token exchange error...
                }
            }
        }
        // ... other auth endpoints
    }
}
```

### Token Validation Strategy

The server implements a dual token validation approach:

```rust
async fn validate_token(&self, token: &str) -> AuthResult<Claims> {
    // Determine token type: JWT has exactly 3 parts separated by dots
    if self.is_jwt_token(token) {
        println!("Token appears to be JWT format - using JWT validation");
        self.validate_jwt_token(token).await
    } else {
        println!("Token appears to be opaque format - using token introspection");
        self.introspect_access_token(token).await
    }
}

pub fn is_jwt_token(&self, token: &str) -> bool {
    // JWT tokens have exactly 3 non-empty parts separated by dots
    let parts: Vec<&str> = token.split('.').collect();
    parts.len() == 3 && parts.iter().all(|part| !part.is_empty())
}
```

**JWT Validation** (for ID tokens):
```rust
async fn validate_jwt_token(&self, token: &str) -> AuthResult<Claims> {
    // Decode JWT header to get key ID
    let header = decode_header(token)?;
    let kid = header.kid.ok_or_else(|| AuthError::InvalidConfig("JWT missing key ID".to_string()))?;

    // Get JWKS and find matching key
    let jwks = self.get_jwks().await?;
    let jwk = jwks.keys.iter().find(|key| key.kid == kid)
        .ok_or_else(|| AuthError::JwksKeyNotFound(kid.clone()))?;

    // Convert JWK to decoding key and validate
    let decoding_key = self.jwk_to_decoding_key(jwk)?;
    let discovery = self.discover_oidc_config().await?;
    let mut validation = Validation::new(Algorithm::RS256);
    validation.set_issuer(&[&discovery.issuer]);
    validation.validate_aud = false; // Don't require audience for client_credentials

    let token_data = decode::<Claims>(token, &decoding_key, &validation)?;
    Ok(token_data.claims)
}
```

**Token Introspection** (for opaque access tokens):
```rust
pub async fn introspect_access_token(&self, token: &str) -> AuthResult<Claims> {
    let introspection_url = format!("{}/api/oidc/introspect", self.config.authelia_base_url);

    let introspection_request = IntrospectionRequest {
        token: token.to_string(),
        client_id: self.config.client_id.clone(),
        client_secret: self.config.client_secret.clone(),
    };

    let response = self.client
        .post(&introspection_url)
        .form(&introspection_request)
        .send()
        .await?;

    let introspection: IntrospectionResponse = response.json().await?;

    if !introspection.active {
        return Err(AuthError::TokenInactive);
    }

    // Convert introspection response to Claims
    let claims = Claims {
        sub: introspection.sub.unwrap_or_else(|| "unknown".to_string()),
        exp: introspection.exp.unwrap_or(0) as usize,
        iat: introspection.iat.unwrap_or(0) as usize,
        iss: introspection.iss.unwrap_or_else(|| discovery.issuer.clone()),
        aud: introspection.aud.unwrap_or_else(|| self.config.client_id.clone()),
        email: None,
        groups: None,
        preferred_username: introspection.username,
    };

    Ok(claims)
}
```

This dual approach correctly handles both JWT ID tokens and Authelia's opaque access tokens.

## Ansible Deployment

### Role Architecture

The `goldentooth.setup_mcp_server` role provides automated deployment:

```
ansible/roles/goldentooth.setup_mcp_server/
├── defaults/main.yaml     # Default configuration variables
├── tasks/main.yaml        # Deployment tasks
├── templates/
│   └── goldentooth-mcp.service.j2  # Systemd service template
├── handlers/main.yaml     # Service management handlers
└── CLAUDE.md             # Role documentation
```

### Automatic Release Management

The role automatically fetches the latest release from GitHub:

```yaml
- name: Get latest release information from GitHub
  uri:
    url: "{{ mcp_server_github_api_url }}/releases/latest"
    method: GET
    return_content: true
  register: github_release
  delegate_to: localhost
  run_once: true

- name: Determine architecture-specific binary name
  set_fact:
    mcp_binary_name: >-
      {%- if ansible_architecture == 'x86_64' -%}
      goldentooth-mcp-x86_64-linux
      {%- elif ansible_architecture == 'aarch64' -%}
      goldentooth-mcp-aarch64-linux
      {%- else -%}
      goldentooth-mcp-x86_64-linux
      {%- endif -%}

- name: Download latest MCP server binary
  get_url:
    url: "{{ mcp_binary_url }}"
    dest: "{{ mcp_server_install_dir }}/{{ mcp_server_binary_name }}"
    mode: '0755'
    owner: root
    group: root
  notify: restart goldentooth-mcp
```

This approach ensures nodes always run the latest release with appropriate architecture support.

### Systemd Service Configuration

The service template includes comprehensive security hardening:

```ini
[Unit]
Description=Goldentooth MCP Server
Documentation=https://github.com/{{ mcp_server_github_repo }}
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User={{ mcp_server_user }}
Group={{ mcp_server_group }}
ExecStart={{ mcp_server_install_dir }}/{{ mcp_server_binary_name }} --http
Environment=MCP_HTTP_MODE=1
Environment=MCP_PORT={{ mcp_server_port }}
Environment=OAUTH_CLIENT_ID={{ mcp_server_oauth_client_id }}
Environment=OAUTH_CLIENT_SECRET={{ secret_vault.authelia.oidc.mcp_secret }}
Environment=AUTHELIA_BASE_URL={{ mcp_server_authelia_base_url }}
Environment=OAUTH_REDIRECT_URI={{ mcp_server_oauth_redirect_uri }}

# Security hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths={{ mcp_server_working_dir }}

# Resource limits
MemoryLimit={{ mcp_server_memory_limit }}
CPUQuota={{ mcp_server_cpu_quota }}

[Install]
WantedBy=multi-user.target
```

Key security features:
- **Dedicated User**: Runs under `mcp` user with minimal privileges
- **Filesystem Protection**: Read-only system, private temp directory
- **Resource Limits**: Memory and CPU quotas prevent resource exhaustion
- **No New Privileges**: Prevents privilege escalation

## Development and Testing

### Comprehensive Test Suite

The MCP server includes extensive testing:

**Unit Tests** (src/lib.rs, src/service.rs):
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_service_creation() {
        let service = GoldentoothService::new();
        assert_eq!(service.get_info().server_info.name, "goldentooth-mcp");
    }

    #[tokio::test]
    async fn test_tools_list() {
        let service = GoldentoothService::new();
        // Test tool enumeration...
    }
}
```

**Integration Tests** (tests/):
- `complete_auth_flow_test.rs`: End-to-end OAuth flow testing
- `oauth_integration_test.rs`: OAuth client integration
- `jwt_validation_test.rs`: JWT token validation
- `token_introspection_test.rs`: Token introspection testing
- `certificate_security_test.rs`: TLS certificate validation
- `dos_protection_test.rs`: DoS protection mechanisms
- `secret_disclosure_test.rs`: Security token handling

**Security Tests**:
```rust
#[tokio::test]
async fn test_request_size_limit() {
    let service = GoldentoothService::new();
    let server = HttpServer::new(service, None);

    // Test oversized request
    let large_request = "x".repeat(2 * 1024 * 1024); // 2MB
    let result = server.handle_request_for_test("POST", &large_request, None).await;

    assert!(result.is_err());
    assert!(result.unwrap_err().contains("Request body too large"));
}

#[tokio::test]
async fn test_html_escaping() {
    let malicious_input = "<script>alert('XSS')</script>";
    let escaped = html_escape(malicious_input);
    assert_eq!(escaped, "&lt;script&gt;alert(&#x27;XSS&#x27;)&lt;/script&gt;");
}
```

### Cross-Compilation Support

The project supports cross-compilation for Raspberry Pi deployment:

```bash
# Install cross-compilation target
rustup target add aarch64-unknown-linux-gnu

# Cross-compile for Raspberry Pi
cargo build --release --target aarch64-unknown-linux-gnu

# CI/CD builds both architectures
# - goldentooth-mcp-x86_64-linux
# - goldentooth-mcp-aarch64-linux
```

### Development Workflow

```bash
# Local development with authentication disabled
cargo run

# Test with authentication enabled
OAUTH_CLIENT_SECRET=test-secret cargo run

# Run all tests
cargo test

# Format code
rustfmt --edition=2024 src/*.rs

# Lint with warnings as errors
cargo clippy -- -D warnings

# Test specific functionality
cargo test test_http_server_initialize -- --nocapture
```

## Performance and Monitoring

### Resource Utilization

The MCP server is designed for efficiency on Raspberry Pi hardware:

- **Memory**: ~10-15MB RSS under normal load
- **CPU**: <1% utilization for typical requests
- **Network**: Minimal bandwidth requirements
- **Startup Time**: ~100ms cold start

### Health Monitoring

Multiple monitoring endpoints provide observability:

**Health Check**:
```http
GET /health HTTP/1.1
Host: mcp.services.goldentooth.net

HTTP/1.1 200 OK
Content-Type: application/json

{"status":"healthy","service":"goldentooth-mcp"}
```

**Service Info**:
```http
POST /mcp/request HTTP/1.1
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "method": "server/get_info",
  "id": 1
}
```

**Metrics Integration**: The service logs are automatically collected by our Vector/Loki stack for centralized monitoring.

## Operational Procedures

### Deployment Commands

```bash
# Deploy MCP server across cluster
goldentooth setup_mcp_server

# Deploy to specific nodes
goldentooth setup_mcp_server --limit allyrion,bettley

# Check service status
goldentooth command mcp "systemctl status goldentooth-mcp"

# View service logs
goldentooth command mcp "journalctl -u goldentooth-mcp -f"

# Restart service
goldentooth command_root mcp "systemctl restart goldentooth-mcp"
```

### Configuration Management

Environment variables are managed through Ansible vault:

```yaml
# In group_vars/all/vault (encrypted)
secret_vault:
  authelia:
    oidc:
      mcp_secret: "real-oauth-client-secret"

# In defaults/main.yaml
mcp_server_oauth_client_id: "goldentooth-mcp"
mcp_server_authelia_base_url: "https://auth.services.goldentooth.net"
mcp_server_oauth_redirect_uri: "https://mcp.services.goldentooth.net/callback"
mcp_server_port: 8085
```

### Troubleshooting

**Authentication Issues**:
```bash
# Check OAuth client configuration in Authelia
goldentooth command authelia "grep -A 10 'goldentooth-mcp' /etc/authelia/configuration.yml"

# Test token introspection endpoint
curl -X POST https://auth.services.goldentooth.net/api/oidc/introspect \
  -d "token=test-token&client_id=goldentooth-mcp&client_secret=secret"

# Verify MCP server can reach Authelia
goldentooth command mcp "curl -k https://auth.services.goldentooth.net/.well-known/openid-configuration"
```

**Network Connectivity**:
```bash
# Check MCP server listening
goldentooth command mcp "netstat -tulpn | grep 8085"

# Test internal connectivity
goldentooth command mcp "curl -k http://localhost:8085/health"

# Verify external access through load balancer
curl https://mcp.services.goldentooth.net/health
```

**Certificate Issues**:
```bash
# Check cluster CA installation
goldentooth command mcp "ls -la /etc/ssl/certs/goldentooth.pem"

# Test TLS connectivity to Authelia
goldentooth command mcp "openssl s_client -connect auth.services.goldentooth.net:443 -servername auth.services.goldentooth.net"
```
