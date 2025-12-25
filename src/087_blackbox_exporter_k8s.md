# Prometheus Blackbox Exporter 2

Way back in entry 053, I set up Blackbox Exporter via Ansible on bare metal. That was a different era – before Talos, before FluxCD, before I nuked everything and rebuilt it properly. Time to bring synthetic monitoring back, but this time the Kubernetes way.

## Why Blackbox Monitoring?

I've got whitebox monitoring covered: Prometheus scrapes node-exporter, kube-state-metrics, application `/metrics` endpoints. I can see if pods are running, if CPU is spiking, if memory is tight.

But none of that tells me: **can I actually reach my websites?**

Enter blackbox monitoring. Instead of asking "is the service running?", we ask "does it work from the outside?" – like an actual user would. Blackbox Exporter makes HTTP requests to URLs and reports whether they succeeded, how long they took, what status code came back.

## The Targets: External GitHub Pages Sites

This isn't about monitoring cluster services (though I could). I want to monitor two external sites:

- `https://goldentooth.net/` – the main site
- `https://clog.goldentooth.net/` – this very journal you're reading

Both are hosted on GitHub Pages. I don't control their infrastructure. GitHub handles the TLS certs, the CDN, all of it. But I still want to know when they're down – partly for awareness, partly so I can feel smug when GitHub has issues instead of wondering if I broke something.

## The GitOps Structure

Created a new infrastructure component:

```
gitops/infrastructure/prometheus-blackbox-exporter/
├── kustomization.yaml
├── release.yaml       # HelmRelease
├── probes.yaml        # Probe CRD (what to monitor)
└── alerts.yaml        # PrometheusRule (when to alert)
```

### The HelmRelease

Pretty minimal. The key piece is the module configuration – this defines *how* to probe:

```yaml
config:
  modules:
    http_2xx:
      prober: http
      timeout: 10s
      http:
        valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
        valid_status_codes: [200, 201, 202, 203, 204, 301, 302]
        method: GET
        follow_redirects: true
        preferred_ip_protocol: ip4
```

The `http_2xx` module says: make an HTTP GET, follow redirects, accept any 2xx or redirect status as success. Simple.

### The Probe CRD

This is where Prometheus Operator shines. Instead of manually configuring Prometheus with relabeling rules, I just declare what I want:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Probe
metadata:
  name: external-websites
  namespace: monitoring
spec:
  interval: 60s
  module: http_2xx
  prober:
    url: prometheus-blackbox-exporter.monitoring.svc.cluster.local:9115
  targets:
    staticConfig:
      static:
        - https://goldentooth.net/
        - https://clog.goldentooth.net/
      labels:
        environment: external
        probe_type: website
```

Every 60 seconds, Prometheus will ask Blackbox to probe both URLs. The Operator handles all the plumbing.

### Alerting Rules

Since these are GitHub Pages sites, I don't need certificate expiry warnings (GitHub handles that). Just two alerts:

```yaml
- alert: WebsiteDown
  expr: probe_success == 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Website {{ $labels.instance }} is down"

- alert: WebsiteSlow
  expr: probe_duration_seconds > 10
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Website {{ $labels.instance }} is slow"
```

The `for: 5m` clause is the Prometheus equivalent of "X failures before alerting" – with a 60-second probe interval, 5 minutes means roughly 5 consecutive failures. No flapping alerts from transient network blips.

Also added a meta-alert for when the blackbox exporter itself is broken:

```yaml
- alert: BlackboxProbeFailed
  expr: up{job="probe"} == 0
  for: 5m
```

Because what good is monitoring if you don't monitor your monitoring?

## Bonus: Exposing Prometheus UI

While I was in there, I realized I'd never exposed the Prometheus UI itself. Grafana was accessible via LoadBalancer, but Prometheus wasn't. Added the service config:

```yaml
prometheus:
  service:
    type: LoadBalancer
    annotations:
      metallb.io/address-pool: default
      external-dns.alpha.kubernetes.io/hostname: prometheus.goldentooth.net
```

Now I can poke around at `prometheus.goldentooth.net` to see raw metrics, check which alerts are registered, debug scrape targets. Much nicer than port-forwarding every time.

## Verification

After Flux reconciled everything:

```bash
$ kubectl get pods -n monitoring | grep blackbox
prometheus-blackbox-exporter-xxx   1/1     Running

$ kubectl get probes -n monitoring
NAME                AGE
external-websites   5m
```

In Prometheus UI → Status → Rules, the `blackbox-exporter` group shows up with all three alerts.

Query `probe_success` and both URLs show `1`. Query `probe_duration_seconds` and GitHub Pages responds in ~200ms. Not bad.
