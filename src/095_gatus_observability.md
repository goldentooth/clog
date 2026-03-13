# Gatus: Metrics, Alerts, and Shiny Badges

## The Status Page That Could Do More

Last session I replaced Uptime Kuma with [Gatus](https://github.com/TwiN/gatus) — a declarative status page that lives entirely in a ConfigMap instead of a click-around web UI. 14 endpoints, health checks, a public status page at `status.goldentooth.net`, done.

But Gatus was sitting there like a security camera that records to `/dev/null`. It knew when things were broken, it just... didn't tell anyone. And it was generating internal metrics nobody was scraping. Time to fix both of those things, plus add some vanity badges to the GitHub profile because why not.

## Prometheus Metrics

Gatus has a built-in Prometheus metrics endpoint. You turn it on with a single line:

```yaml
metrics: true
```

That's it. That's the config. Gatus starts exposing `/metrics` on the same port as the UI (8080), and you get counters like `gatus_results_total` broken down by endpoint, success/failure, and HTTP status code. Free time-series data about every health check, forever, bolted straight into the existing Prometheus/Grafana stack.

The ServiceMonitor was straightforward:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: gatus
  namespace: gatus
  labels:
    release: monitoring-kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: gatus
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

...except I initially got the label wrong. Used `release: kube-prometheus-stack` because that's what the Blackbox Exporter's ServiceMonitor uses. Prometheus ignored it completely. No errors, no warnings, just silence — the Prometheus operator's favorite way to tell you you're wrong.

Turns out the _actual_ selector the operator is looking for is `release: monitoring-kube-prometheus-stack`. The Blackbox Exporter's ServiceMonitor had the wrong label too, it was just getting scraped via a different mechanism. I found the truth by checking what Prometheus actually expects:

```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}'
```

```json
{"matchLabels":{"release":"monitoring-kube-prometheus-stack"}}
```

Fixed the label. Prometheus immediately picked it up. `gatus_results_total` started flowing. I could now query things like "show me the 99th percentile response time for the Kubernetes API health endpoint over the last 24 hours" and get an actual answer.

I also had to add an `app: gatus` label to the Service itself, since the ServiceMonitor needs to select the Service by label and it didn't have one. The kind of thing you miss when you write manifests by hand instead of using Helm charts that wire everything together automatically.

## ntfy Alerting

Gatus supports ntfy as a native alerting provider. We already have ntfy running in the cluster (deployed back in entry 090), and Alertmanager is already sending Prometheus alerts to it via webhook. Adding Gatus alerts to the same notification pipe was trivial:

```yaml
alerting:
  ntfy:
    url: http://ntfy.ntfy.svc.cluster.local
    topic: gatus-alerts
    priority: 3
    default-alert:
      failure-threshold: 3
      success-threshold: 2
      send-on-resolved: true
```

Separate topic from Alertmanager (`gatus-alerts` vs `cluster-alerts`) so I can tell at a glance whether it's a Prometheus rule firing or a Gatus endpoint going dark. Priority 3 (default) because these are "hey, something's not responding" checks, not "the cluster is on fire" alerts.

The `default-alert` block means I don't have to repeat the thresholds on every endpoint. Each endpoint just needs:

```yaml
alerts:
  - type: ntfy
```

Added that to all 12 internal endpoints — everything in the `cluster`, `infrastructure`, and `apps` groups. The two external endpoints (`goldentooth.net` and `clog.goldentooth.net`) don't get alerts because they're hosted on GitHub Pages and Cloudflare, and there's not a lot I can do if GitHub goes down besides join the collective screaming on social media.

### The Circularity Problem

There is one entertaining little issue: Gatus monitors ntfy and alerts via ntfy. If ntfy goes down, Gatus will dutifully try to send a "ntfy is down" notification to... ntfy. Which is down. So I'll never get that particular alert.

This is fine. Alertmanager independently monitors ntfy via Prometheus metrics, and Alertmanager has its own webhook to ntfy, so... wait. That's the same problem. If ntfy is truly dead, neither system can notify me through the ntfy channel.

The real safety net is that Prometheus also fires `PodCrashLooping` and `PodNotReady` alerts, which would go through Alertmanager's webhook to ntfy. So we're still circular. In practice, ntfy has never gone down, and if it does, I'll notice when my phone stops buzzing about routine things. The absence of notifications _is_ the notification. Zen monitoring.

## Status Badges

Gatus exposes SVG badges for every endpoint — health status and uptime percentage. The URLs follow a predictable pattern:

```
https://status.goldentooth.net/api/v1/endpoints/{key}/health/badge.svg
https://status.goldentooth.net/api/v1/endpoints/{key}/uptimes/7d/badge.svg
```

The `{key}` is `{group}_{name}` with special characters replaced by hyphens. So `infrastructure/Kubernetes API` becomes `infrastructure_kubernetes-api`, `cluster/Step CA` becomes `cluster_step-ca`, etc.

Added a badge table to the GitHub org profile README template:

```markdown
| Service        | Health         | Uptime (7d)    |
| -------------- | -------------- | -------------- |
| Kubernetes API | ![Health](...) | ![Uptime](...) |
| Prometheus     | ![Health](...) | ![Uptime](...) |
| Grafana        | ![Health](...) | ![Uptime](...) |
| Step CA        | ![Health](...) | ![Uptime](...) |
```

Four key services. Kubernetes API because it's the beating heart. Prometheus and Grafana because they're the eyes. Step CA because it's the PKI root and if it's down, certificates stop renewing. Anyone visiting the GitHub org page now gets live health and uptime badges, which is either impressively professional or deeply unnecessary for a home cluster. Both, probably.

## The Auto-Discovery Detour

I should mention the detour that didn't work. Before all of this, I spent a while trying to set up Kubernetes auto-discovery in Gatus — the idea being that Gatus would watch the Kubernetes API for annotated Services and automatically create endpoints for them. No more manual ConfigMap edits when you deploy a new app.

I deployed RBAC manifests, added annotations to services, wrote a `kubernetes:` config block. Everything looked right. Gatus started up fine.

Nothing happened. No auto-discovered endpoints. No errors. No warnings.

After some digging, I discovered that Gatus doesn't actually have built-in Kubernetes auto-discovery. The `kubernetes:` config block was silently ignored — Gatus doesn't validate unknown top-level keys. The feature I was trying to use simply doesn't exist. There are third-party tools and forks that add it, but upstream Gatus is strictly "you declare your endpoints in the config file."

Cleaned up all the auto-discovery artifacts — removed the RBAC, removed the annotations, removed the dead config block. Back to manual endpoint declaration, which is honestly fine for 14 endpoints. If the cluster grows to 50 monitored services, I'll revisit.

## The Result

Gatus now does three things it didn't do before:

1. **Pushes metrics to Prometheus** — `gatus_results_total` and friends, scraped every 30 seconds, available for Grafana dashboards and alerting rules
2. **Sends failure notifications to ntfy** — phone buzzes when an endpoint fails 3 consecutive checks, buzzes again when it recovers
3. **Exposes live status badges** — health and 7-day uptime for key services, embedded in the GitHub org profile

All from a single ConfigMap, a 16-line ServiceMonitor, and a table of Markdown image links. Not bad for an afternoon's work on a status page.
