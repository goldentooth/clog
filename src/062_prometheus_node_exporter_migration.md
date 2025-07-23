# Prometheus Node Exporter Migration: From Kubernetes to Native

## The Problem

While working on Grafana dashboard configuration, I discovered that the node exporter dashboard was completely empty - no metrics, no data, just a sad empty dashboard that looked like it had given up on life.

The issue? Our Prometheus Node Exporter was deployed via Kubernetes and Argo CD, but Prometheus itself was running as a systemd service on `allyrion`. The Kubernetes deployment created a ClusterIP service at `172.16.12.161:9100`, but Prometheus (running outside the cluster) couldn't reach this internal Kubernetes service.

Meanwhile, Prometheus was configured to scrape node exporters directly at each node's IP on port 9100 (e.g., `10.4.0.11:9100`), but nothing was listening there because the actual exporters were only accessible through the Kubernetes service mesh.

## The Solution: Raw-dogging Node Exporter

Time to embrace the chaos and deploy node exporter directly on the nodes as systemd services. Sometimes the simplest solution is the best solution.

### Step 1: Create the Ansible Playbook

First, I created a new playbook to deploy node exporter cluster-wide using the same `prometheus.prometheus.node_exporter` role that HAProxy was already using:

```yaml
# ansible/playbooks/setup_node_exporter.yaml
# Description: Setup Prometheus Node Exporter on all cluster nodes.

- name: 'Setup Prometheus Node Exporter.'
  hosts: 'all'
  remote_user: 'root'
  roles:
    - { role: 'prometheus.prometheus.node_exporter' }
  handlers:
    - name: 'Restart Node Exporter.'
      ansible.builtin.service:
        name: 'node_exporter'
        state: 'restarted'
        enabled: true
```

### Step 2: Deploy via Goldentooth CLI

Thanks to the goldentooth CLI's fallback behavior (it automatically runs Ansible playbooks with matching names), deployment was as simple as:

```bash
goldentooth setup_node_exporter
```

This installed node exporter on all 13 cluster nodes, creating:
- `node-exp` system user and group
- `/usr/local/bin/node_exporter` binary
- `/etc/systemd/system/node_exporter.service` systemd service
- `/var/lib/node_exporter` textfile collector directory

### Step 3: Handle Port Conflicts

The deployment initially failed on most nodes with "address already in use" errors. The Kubernetes node exporter pods were still running and had claimed port 9100.

Investigation revealed the conflict:
```bash
goldentooth command bettley "journalctl -u node_exporter --no-pager -n 10"
# Error: listen tcp 0.0.0.0:9100: bind: address already in use
```

### Step 4: Clean Up Kubernetes Deployment

I removed the Kubernetes deployment entirely:

```bash
# Delete the daemonset and namespace
kubectl delete daemonset prometheus-node-exporter -n prometheus-node-exporter
kubectl delete namespace prometheus-node-exporter

# Delete the Argo CD applications managing this
kubectl delete application prometheus-node-exporter gitops-repo-prometheus-node-exporter -n argocd

# Delete the GitHub repository (to prevent ApplicationSet from recreating it)
gh repo delete goldentooth/prometheus-node-exporter --yes
```

### Step 5: Restart Failed Services

With the port conflicts resolved, I restarted the systemd services:

```bash
goldentooth command bettley,dalt "systemctl restart node_exporter"
```

All nodes now showed healthy node exporter services:
```
● node_exporter.service - Prometheus Node Exporter
     Loaded: loaded (/etc/systemd/system/node_exporter.service; enabled)
     Active: active (running) since Wed 2025-07-23 19:36:30 EDT; 7s ago
```

### Step 6: Reload Prometheus

With native node exporters now listening on port 9100 on all nodes, I reloaded Prometheus to pick up the new targets:

```bash
goldentooth command allyrion "systemctl reload prometheus"
```

Verified metrics were accessible:
```bash
goldentooth command allyrion "curl -s http://10.4.0.11:9100/metrics | grep node_cpu_seconds_total | head -3"
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 1.42238869e+06
```

## The Result

Within minutes, the Grafana node exporter dashboard came alive with beautiful metrics from all cluster nodes. CPU usage, memory consumption, disk I/O, network statistics - everything was flowing perfectly.
