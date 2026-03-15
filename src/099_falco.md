# Falco: Runtime Security for the Bramble

## Why Runtime Security?

I've got observability coming out of my ears at this point. Prometheus scrapes everything that moves, Loki ingests every log line, Tempo traces requests across services, Alloy shuttles telemetry around, Gatus checks endpoints, Blackbox Exporter probes from the outside. I can tell you exactly how many bytes Garage wrote to disk at 3:47 AM last Tuesday. What I *couldn't* tell you is whether something inside a container just read `/etc/shadow` or opened a reverse shell.

Falco fills that gap. It's a CNCF graduated project that uses eBPF to monitor syscalls at the kernel level — every `open()`, `connect()`, `execve()`, `dup()` across every container on every node. The default ruleset catches the classics: sensitive file reads, unexpected network connections, privilege escalation, container escapes, crypto mining processes. It's the security equivalent of "I don't know what I'm looking for, but I'll know it when I see it."

## The Deployment

### Architecture

Falco runs as a DaemonSet — one pod per node. Each pod loads an eBPF probe into the kernel and watches syscalls in real time. Events matching rules get forwarded to Falcosidekick, which fans them out to:

- **Alertmanager** (via v2 API) — for warnings and above, integrating with existing Prometheus alerting
- **ntfy** (via webhook) — for critical events only, because I don't need my phone buzzing every time nic-watchdog pings the gateway

The whole stack:

```
Kernel syscalls → eBPF probe → Falco engine → Rules evaluation
                                                    ↓
                                              Falcosidekick
                                              ↙          ↘
                                    Alertmanager        ntfy (critical only)
                                         ↓
                                    Prometheus/Grafana
```

### Talos + eBPF: A Love Story

Talos Linux has an immutable rootfs. No kernel headers. No `apt install`. No `insmod`. This rules out Falco's traditional kernel module driver entirely — which is fine, because the kernel module approach was always kind of terrifying anyway.

Falco's `modern_ebpf` driver is the answer. It's compiled directly into the Falco binary, so there's no init container downloading drivers, no kernel header matching dance, no "sorry, we don't have a prebuilt probe for your kernel version." The eBPF probe just loads. Talos ships kernel 6.18.x, which is well above the minimum 5.8 requirement for modern eBPF. Every Pi 4B (Cortex-A72) and Pi 5 (Cortex-A76) handles it fine.

```yaml
driver:
  kind: modern_ebpf
  loader:
    enabled: false    # No driver loader needed — probe is built into the binary
```

Two lines of config. That's it. No drama.

### The GitOps Setup

Standard four-file Flux structure in `gitops/infrastructure/falco/`:

```
falco/
├── kustomization.yaml
├── namespace.yaml          # Privileged PSA (needs host-level eBPF access)
├── repository.yaml         # HelmRepository → falcosecurity.github.io/charts
└── release.yaml            # HelmRelease with Falco + Falcosidekick config
```

Key values:

```yaml
# Modern eBPF, no loader
driver:
  kind: modern_ebpf
  loader:
    enabled: false

# JSON output for Loki, ISO timestamps
falco:
  json_output: true
  json_include_output_property: true
  json_include_tags_property: true
  time_format_iso_8601: true
  log_syslog: false       # Talos has no syslog
  http_output:
    enabled: true
    url: http://falco-falco-falcosidekick.falco.svc:2801

# Prometheus integration
serviceMonitor:
  create: true
  labels:
    release: monitoring-kube-prometheus-stack

# Falcosidekick sub-chart
falcosidekick:
  enabled: true
  config:
    alertmanager:
      hostport: http://monitoring-kube-prometheus-alertmanager.monitoring.svc:9093
      endpoint: /api/v2/alerts
      minimumpriority: warning
    webhook:
      address: http://ntfy.ntfy.svc:80/falco
      minimumpriority: critical
```

Resource limits are conservative given the Pi 4B fleet: 50m/256Mi requests, 500m/512Mi limits for Falco; 20m/64Mi requests for sidekick.

## The Debugging Gauntlet

### Bug 1: The Service Name

Falcosidekick deploys as a service, and Falco's `http_output` needs to reach it. I initially configured:

```yaml
url: http://falco-falcosidekick.falco.svc:2801
```

The actual service name:

```yaml
url: http://falco-falco-falcosidekick.falco.svc:2801
```

The Helm chart names the service `{release}-{chart}-falcosidekick`. Since the release is `falco-falco` (Flux prefixes with the HelmRelease name... or something — honestly I've stopped trying to predict Helm naming) the service gets `falco-falco-falcosidekick`. The SeaweedFS filer had the exact same class of bug two hours earlier. I'm beginning to think "guess the Helm service name" should be its own drinking game.

### Bug 2: The DaemonSet Timeout

Helm's `--wait` flag blocks until ALL pods in a release are Ready and up-to-date. When you're rolling a DaemonSet across 16 Raspberry Pi nodes — each one pulling a 40MB container image over the network, starting an eBPF probe, loading rules, and becoming ready — "wait for everything" takes a while. More than 5 minutes. More than 10 minutes.

The first install timed out at 5 minutes (default). Bumped to 10 minutes. Timed out again. The DaemonSet was *actually working* — 15/16 pods ready, just one slow node still pulling the image — but Helm doesn't care about "almost done."

The fix: `disableWait: true` and `timeout: 15m` in the HelmRelease. Helm submits the manifests and returns immediately. The DaemonSet controller handles the rollout at its own pace.

```yaml
install:
  crds: CreateReplace
  disableWait: true
  remediation:
    retries: 3
upgrade:
  crds: CreateReplace
  disableWait: true
  remediation:
    retries: 3
```

After clearing Helm release secrets and doing a fresh install with these settings, it went through cleanly.

### Bug 3: Alertmanager 410 Gone

Falcosidekick's Alertmanager integration was returning `410 Gone`:

```
2026/03/15 05:14:28 [ERROR] : AlertManager - unexpected Response (410)
2026/03/15 05:14:28 [ERROR] : AlertManager - 410 Gone
```

Sidekick defaults to the Alertmanager v1 API, which newer Alertmanager versions have deprecated and removed. One line fix:

```yaml
alertmanager:
  endpoint: /api/v2/alerts
```

After that: `AlertManager - POST OK (200)`. 

## What Falco Sees

Out of the box, Falco immediately started flagging things on the cluster:

**"Contact K8S API Server From Container"** — Garage's tokio workers connecting to the K8s API for peer discovery. Expected behavior, `Notice` priority.

**"Redirect STDOUT/STDIN to Network Connection in Container"** — nic-watchdog's busybox `ping` command. It pings the gateway every 15 seconds to check NIC health. Also expected, also `Notice`.

These are all legitimate activity that the default rules flag at low priority. They won't trigger Alertmanager (set to `warning`+) or ntfy (set to `critical` only), but they'll show up in Loki for forensic analysis. If I wanted to suppress them, I could add exception lists to the Falco rules — but having them in the log is actually useful for establishing a behavioral baseline.

The test I ran — `kubectl run falco-test --image=busybox --rm -it -- cat /etc/shadow` — got caught and forwarded to Alertmanager successfully. Falco saw the sensitive file read, classified it as `Warning`, Sidekick POSTed to Alertmanager v2 API, got a 200. The pipeline works end to end.

## Coverage

Falco is now running on 16 of 17 nodes:

| Nodes | Count | Coverage |
|-------|-------|----------|
| Pi 4B workers | 9 | All covered |
| Pi 4B control plane | 3 | All covered |
| Pi 5 NVMe workers | 4 | All covered |
| x86 GPU (velaryon) | 1 | Not covered (tainted) |

Velaryon has `platform=x86:NoSchedule` and `gpu=true:NoSchedule` taints. Adding Falco tolerations for it would be trivial, but the GPU node runs JupyterLab and gaming containers — probably the node that *most* deserves security monitoring, actually. Something for the future.

## The Stack So Far

Adding Falco to the running tally:

- **Metrics**: Prometheus + Node Exporter + Blackbox Exporter + Grafana
- **Logs**: Loki + Alloy
- **Traces**: Tempo + OpenTelemetry
- **Status**: Gatus
- **Chaos**: Chaos Mesh
- **Security**: Falco + Falcosidekick ← new
- **Notifications**: ntfy

Runtime security was the obvious gap. Now if someone manages to `exec` into a pod and start doing recon, I'll know about it before they do.
