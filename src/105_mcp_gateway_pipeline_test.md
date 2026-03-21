# Exposing the MCP Server: Gateway, TLS, and the 14-Minute Loop

## The Goal

The MCP server has been running on the cluster for a while now, but Claude Code was talking to it via a local stdio process — basically just shelling out to a binary on my machine. That's fine for development, but the whole point of deploying this thing to the bramble was to have it *on* the bramble. Running on Pi hardware. Talking to cluster APIs. Being a real service.

So: expose it through the gateway at `mcp.goldentooth.net`, configure Claude Code to connect to it over SSE, and then — because why not — test the entire CI/CD pipeline end-to-end by adding a new tool and timing how long it takes from `git push` to "Claude Code can call the new function."

## Gateway Route

The MCP server was already deployed in the `goldentooth-mcp` namespace with a Service on port 8080. Getting it onto the gateway was the easy part — just another HTTPRoute:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: goldentooth-mcp
  namespace: goldentooth-mcp
spec:
  parentRefs:
    - name: goldentooth
      namespace: gateway
      sectionName: https
  hostnames:
    - mcp.goldentooth.net
  rules:
    - backendRefs:
        - name: goldentooth-mcp
          port: 8080
```

Added `mcp.goldentooth.net` to the gateway Certificate's `dnsNames` list so cert-manager would include it in the next TLS cert. Quick `curl` confirmed it was live:

```
$ curl -ks -H "Accept: text/event-stream" https://mcp.goldentooth.net/sse
Bad Request: Session ID is required
```

That's the correct response for an SSE endpoint with no session. We're in business.

## Configuring Claude Code: The TLS Saga

This is where it got annoying.

Claude Code supports remote MCP servers over HTTP/SSE. The config is straightforward — run `claude mcp add --transport http goldentooth-mcp https://mcp.goldentooth.net/sse` and it writes the config to `~/.claude.json`. Simple.

Except it couldn't connect. `Failed to connect` on every attempt.

The issue: TLS. The gateway serves certs signed by the Step-CA intermediate, and Claude Code (being a Node.js app) doesn't trust my private CA by default. You tell Node about custom CAs via `NODE_EXTRA_CA_CERTS`. Added that to `.claude/settings.local.json`:

```json
{
  "env": {
    "NODE_EXTRA_CA_CERTS": "/Users/nathan/goldentooth_ca.crt"
  }
}
```

Still didn't work. Because `~/root_ca.crt` was *the wrong root CA*.

Turns out I had two different PKIs floating around. The old `root_ca.crt` was from the Raspbian era — before the Talos migration, before Step-CA was even running on the cluster. Different org name (`goldentooth` vs `Goldentooth CA`), different key, completely unrelated cert. The gateway certs are signed by the Step-CA intermediate, whose root is `Goldentooth CA Root CA`. I found the correct root by pulling it from the ClusterIssuer's `caBundle`:

```
$ kubectl get clusterissuer -o yaml | grep "caBundle:" | awk '{print $2}' | base64 -d > ~/goldentooth_ca.crt
$ openssl x509 -in ~/goldentooth_ca.crt -noout -subject
subject=O=Goldentooth CA, CN=Goldentooth CA Root CA
```

With the right root CA, everything connected instantly:

```
$ NODE_EXTRA_CA_CERTS=~/goldentooth_ca.crt claude mcp list
goldentooth-mcp: https://mcp.goldentooth.net/sse (HTTP) - ✓ Connected
```

### Lessons in MCP Config

I also learned some things the hard way about how Claude Code discovers MCP servers:

1. **`claude mcp add` writes to `~/.claude.json`**, not to the `.mcp.json` files. The `.mcp.json` files are a separate mechanism. I created configs in three different places before figuring this out.
2. **The transport type is `http`, not `sse`.** Claude Code uses `http` for all remote MCP servers regardless of whether they use SSE or Streamable HTTP.
3. **Project-scoped `.mcp.json` files need explicit approval** via `enableAllProjectMcpServers: true` in settings. Makes sense — you don't want random repos auto-connecting to arbitrary MCP servers.

## The Pipeline Test

With the MCP server live on the gateway, I wanted to test the full CI/CD loop. Added a simple `list_nodes` tool to the MCP server — static data returning all 16 bramble nodes with hardware info. Nothing fancy, but enough to be clearly visible as a new tool.

The pipeline:

```
git push (GitHub)
  → Forgejo mirror sync (every 5m)
    → Forgejo Actions (Kaniko ARM64 build)
      → Registry push
        → Flux ImageRepository scan (every 1m)
          → ImagePolicy selects new tag
            → ImageUpdateAutomation commits to gitops
              → Flux reconciles → new pod
```

### The Timeline

| Event                    | Time     | Delta     |
| ------------------------ | -------- | --------- |
| `git push`               | 14:28:55 | —         |
| Forgejo mirror sync      | 14:34:30 | +5:35     |
| Kaniko build complete    | 14:40:47 | +6:17     |
| Flux ImagePolicy updated | 14:41:35 | +0:48     |
| New pod running          | 14:43:06 | +1:29     |
| **Total**                |          | **14:11** |

The mirror sync was the first bottleneck at 5 minutes (I got impatient and triggered it manually via the API). The Kaniko build was the real bottleneck at 6+ minutes — compiling a Rust release binary with musl static linking on a Raspberry Pi is just slow, even though it's technically a native build (ARM64 on ARM64). The Dockerfile has cross-compilation tooling installed because it was written to also work from x86 runners, but in this case it's all native. The Flux automation (scan → policy → git commit → reconcile → deploy) was impressively quick at under 2.5 minutes.

### MCP Tool Discovery

Once the new pod was running, the interesting question was: does Claude Code see the new `list_nodes` tool?

**No.** Not automatically.

MCP tools are discovered at session start. The SSE connection established a session with the old pod, and when that pod died during the deploy, the session died with it. Tool calls returned `Session not found`. The tool list was stale — still showing only `get_version` from the original connection.

Running `/mcp` to reconnect fixed it. Claude Code re-initialized the SSE connection, re-fetched the tool list, and `list_nodes` appeared. Called it successfully — all 16 nodes returned.

There's a known issue (GitHub #30224) where Claude Code reconnects after a server restart but doesn't re-send the `initialize` handshake, leaving the session stuck. The `/mcp` command is the workaround. Not ideal, but workable. The MCP spec does define a `notifications/tools/list_changed` notification, but Claude Code doesn't handle it yet.

## Current State

The MCP server is live at `mcp.goldentooth.net` and Claude Code connects to it over SSE through the gateway. New tools deployed to the cluster appear after a `/mcp` reconnect. The full push-to-deploy pipeline takes about 14 minutes, dominated by the ARM64 cross-compile.

The ghost of `root_ca.crt` from the Raspbian days has been replaced by the correct Step-CA root. One fewer artifact from the before-times cluttering up my home directory.
