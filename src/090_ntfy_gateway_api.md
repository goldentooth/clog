# ntfy, Gateway API, and the Three-Day Cert Outage

I started this session with a vague "how's everything going?" and ended up deploying a push notification server, building an HTTPS gateway for the entire cluster, and discovering that certificate issuance had been silently broken for three days. The usual.

## Push Notifications with ntfy

The cluster had Prometheus, Grafana, Loki, Alloy, Tempo — basically the entire CNCF observability buffet. What it didn't have was any way to *tell me* when something was wrong. Alertmanager was running but had no receivers configured. Just collecting alerts and holding them, like a jar of screams on a shelf.

I deployed [ntfy](https://ntfy.sh/), a lightweight HTTP-based pub/sub notification server. It's beautifully simple: POST to a topic, subscribers get notified. No OAuth dance, no webhook signing secrets, no "please configure your SMTP relay." Just HTTP.

The deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ntfy
  namespace: ntfy
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: ntfy
          image: binwiederhier/ntfy:v2.11.0
          args: ["serve"]
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 128Mi
```

Tiny. Runs on practically nothing. The config is a ConfigMap with `server.yml`:

```yaml
base-url: "https://ntfy.goldentooth.net"
cache-file: "/var/cache/ntfy/cache.db"
cache-duration: "12h"
behind-proxy: true
```

Then I wired Alertmanager to POST to ntfy:

```yaml
alertmanager:
  config:
    route:
      receiver: ntfy
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
    receivers:
      - name: ntfy
        webhook_configs:
          - url: 'http://ntfy.ntfy.svc.cluster.local/cluster-alerts'
            send_resolved: true
```

## PrometheusRules: Things Worth Screaming About

With the notification pipe in place, I needed alerts. I created 11 rules across three groups:

**Node health**: NodeDown (5m), NodeHighCPU (>90%, 10m), NodeHighMemory (>90%, 10m), NodeDiskPressure (>85%, 5m), NodeDiskCritical (>95%, 5m), NodeNotReady (5m).

**Kubernetes workloads**: PodCrashLooping (>5 restarts in 15m), PodNotReady (10m), DeploymentReplicasMismatch (10m), PVCAlmostFull (>85%).

**Observability health**: PrometheusStorageFilling (>80%), LokiStorageFilling (>80%).

All labeled `release: kube-prometheus-stack` so the operator picks them up. I also enabled Hubble's ServiceMonitor and Grafana dashboards in Cilium's HelmRelease — the metrics were being generated but nobody was scraping them. Free observability, just sitting on the floor.

## The HTTPS Problem

ntfy was up, alerts were flowing, everything was great. Then I tried to enable browser notifications and discovered that the Push API requires HTTPS. And our services were all plain HTTP behind MetalLB LoadBalancers. Each service had its own IP address from the MetalLB pool — functional, but unencrypted and burning through IPs.

I decided to fix this properly: a single HTTPS gateway with TLS termination, hostname-based routing, and cert-manager integration with our existing Step-CA PKI.

## Cilium Gateway API

Cilium 1.16 has built-in Gateway API support. One Gateway resource, multiple HTTPRoutes, TLS termination, the works. No need for nginx-ingress or Traefik or any of the other usual suspects.

First, I enabled it in the Cilium HelmRelease:

```yaml
gatewayAPI:
  enabled: true
```

Then created the Gateway:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: goldentooth
  namespace: gateway
  annotations:
    external-dns.alpha.kubernetes.io/hostname: >-
      grafana.goldentooth.net, prometheus.goldentooth.net,
      ntfy.goldentooth.net, hubble.goldentooth.net,
      httpbin.goldentooth.net, jupyterlab.goldentooth.net,
      chaos-center.goldentooth.net, tekton-dashboard.goldentooth.net
spec:
  gatewayClassName: cilium
  listeners:
    - name: http
      port: 80
      protocol: HTTP
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: gateway-tls
```

Each service gets an HTTPRoute:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: ntfy
  namespace: ntfy
spec:
  parentRefs:
    - name: goldentooth
      namespace: gateway
      sectionName: https
  hostnames:
    - ntfy.goldentooth.net
  rules:
    - backendRefs:
        - name: ntfy
          port: 80
```

Plus a global HTTP→HTTPS redirect on the http listener. I switched eight services from LoadBalancer to ClusterIP: ntfy, grafana, prometheus, hubble-ui, httpbin, jupyterlab, litmus frontend, and tekton-dashboard. The long-term services (step-ca, seaweedfs, docker-registry, netboot) keep their dedicated IPs since they're accessed by non-HTTP clients or pre-boot infrastructure.

A `ReferenceGrant` allows HTTPRoutes in other namespaces to reference the Gateway:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-routes-to-gateway
  namespace: gateway
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: ntfy
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: monitoring
    # ... etc
  to:
    - group: gateway.networking.k8s.io
      kind: Gateway
```

## Three Bugs In a Trenchcoat

None of this worked on the first try. Or the second. There were three separate issues, stacked on top of each other like a debugging matryoshka.

### Bug 1: The Missing GRPCRoute CRD

After enabling Gateway API and restarting the Cilium operator, the GatewayClass showed `Pending: Waiting for controller`. The operator logs revealed:

```
level=error msg="Required GatewayAPI resources are not found"
  error="customresourcedefinitions.apiextensions.k8s.io
  \"grpcroutes.gateway.networking.k8s.io\" not found"
```

Cilium 1.16 requires the **experimental** channel Gateway API CRDs, not just the standard ones. The standard install gives you GatewayClass, Gateway, HTTPRoute, and ReferenceGrant. Cilium also demands GRPCRoute and TLSRoute, which live in the experimental channel. I'd already installed TLSRoute but missed GRPCRoute.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/\
  config/crd/experimental/gateway.networking.k8s.io_grpcroutes.yaml
```

Then another operator restart. This time: Gateway Accepted, Gateway Programmed, IP assigned. 10.4.11.1. Beautiful.

### Bug 2: cert-manager-approver-policy and the Phantom CA

The Gateway had an IP but no TLS cert. The Certificate resource was stuck — the CertificateRequest `gateway-tls-1` had been created but showed no conditions at all. Not approved, not denied. Just... nothing.

The cert-manager-approver-policy pod had been crash-looping for **ten days** (2,069 restarts). The error:

```
"Failed to generate serving certificate"
  err="failed verifying CA keypair: tls: failed to find any PEM data in certificate input"
```

The TLS secret for the approver's webhook (`cert-manager-approver-policy-tls`) was present with valid-looking data: `ca.crt`, `tls.crt`, `tls.key`. The CA cert decoded fine with openssl. So what's the problem?

I deleted the pod. The new one came up 1/1 Running. Checked the logs — same error on startup, but the controller started anyway. Workers running, CertificateRequestPolicy `approve-step-ca-requests` showing Ready. But still not processing any CertificateRequests. Zero. Not a single one.

Then I deleted the TLS secret and restarted the pod. And the *real* error appeared:

```
"error ensuring CA"
  err="secrets is forbidden: User
  \"system:serviceaccount:cert-manager:cert-manager-approver-policy\"
  cannot create resource \"secrets\" in API group \"\" in the namespace \"cert-manager\""
```

The RBAC Role had `create` permission on secrets, but scoped to `resourceNames: [cert-manager-approver-policy-tls]`. Here's the thing about Kubernetes RBAC: `resourceNames` restrictions don't work with `create` because the resource doesn't exist yet at authorization time. There's nothing to match the name against. The original secret was created by the Helm install, and after that the controller only needed `update`. But once the secret was gone, the controller couldn't recreate it.

The fix was dumb and effective: create an empty stub secret, then restart the pod:

```bash
kubectl create secret generic cert-manager-approver-policy-tls -n cert-manager
kubectl delete pod -n cert-manager cert-manager-approver-policy-d8df87467-s24fm
```

The controller came up, found the empty secret, and *updated* it with a fresh CA keypair — which it had permission to do. New CA cert, valid for a year. Then I had to update the webhook's `caBundle` to match the new CA:

```bash
NEW_CA=$(kubectl get secret -n cert-manager cert-manager-approver-policy-tls \
  -o jsonpath='{.data.ca\.crt}')
kubectl get validatingwebhookconfiguration cert-manager-approver-policy -o json \
  | jq --arg ca "$NEW_CA" '.webhooks[0].clientConfig.caBundle = $ca' \
  | kubectl apply -f -
```

But the approver *still* wasn't auto-approving CertificateRequests. The controller started, declared itself ready, started workers with `"worker count"=1` — and then sat there doing nothing. I installed `cmctl` and manually approved the stuck requests:

```bash
cmctl approve -n gateway gateway-tls-1
cmctl approve -n cert-test canary-certificate-2483
cmctl approve -n docker-registry registry-tls-139
cmctl approve -n cert-test test-certificate-160
```

All four immediately went to Approved + Ready. The certificate pipeline was working — it was just the approval step that was stuck. Looking at the timeline, the last successful auto-approval was ~7 days ago. Every CR created after that was silently dropped. The canary cert (which renews every few hours) had been quietly failing for days and nobody knew because... we didn't have alerting. Which is what started this whole session.

The cert-manager-approver-policy pod is now healthy and running with a fresh TLS keypair. Whether it'll auto-approve future CRs remains to be seen — the canary cert expires in about two hours, so that'll be the test.

### Bug 3: External-DNS and the Service Annotation Gap

Gateway programmed. Certs issued. HTTPS working via `curl --resolve`. But DNS wasn't resolving. The Gateway had `external-dns.alpha.kubernetes.io/hostname` annotations with all eight hostnames. So why wasn't External-DNS picking them up?

Because External-DNS was configured with `--source=service` only. It watches Services, not Gateway resources. And Cilium, while it helpfully auto-creates a `cilium-gateway-goldentooth` LoadBalancer Service for the Gateway, does *not* propagate annotations from the Gateway to the Service.

The annotation was on the Gateway. External-DNS was watching Services. The auto-created Service had no hostname annotation. Three things that individually made perfect sense and collectively produced silence.

Quick fix — annotate the auto-created Service directly:

```bash
kubectl annotate svc -n gateway cilium-gateway-goldentooth \
  "external-dns.alpha.kubernetes.io/hostname=grafana.goldentooth.net,\
prometheus.goldentooth.net,ntfy.goldentooth.net,hubble.goldentooth.net,\
httpbin.goldentooth.net,jupyterlab.goldentooth.net,\
chaos-center.goldentooth.net,tekton-dashboard.goldentooth.net" \
  "external-dns.alpha.kubernetes.io/ttl=60"
```

Within 60 seconds, External-DNS was creating A records in Route 53:

```
Desired change: CREATE grafana.goldentooth.net A
Desired change: CREATE prometheus.goldentooth.net A
Desired change: CREATE ntfy.goldentooth.net A
Desired change: CREATE hubble.goldentooth.net A
Desired change: CREATE httpbin.goldentooth.net A
Desired change: CREATE jupyterlab.goldentooth.net A
Desired change: CREATE chaos-center.goldentooth.net A
Desired change: CREATE tekton-dashboard.goldentooth.net A
```

All pointing at 10.4.11.1.

## The Final State

Verified with curl:

| Service | Status | Notes |
|---------|--------|-------|
| ntfy.goldentooth.net | 200 | Push notifications |
| grafana.goldentooth.net | 302 | Redirects to /login |
| prometheus.goldentooth.net | 302 | Normal |
| hubble.goldentooth.net | 200 | Network observability |
| httpbin.goldentooth.net | 200 | HTTP testing |
| jupyterlab.goldentooth.net | — | GPU workbench |
| chaos-center.goldentooth.net | — | Litmus chaos |
| tekton-dashboard.goldentooth.net | — | CI/CD |

HTTP requests to port 80 return a 301 redirect to HTTPS. The Gateway has a single MetalLB IP (10.4.11.1) instead of eight separate LoadBalancer IPs. TLS terminates at the gateway with certs from Step-CA, renewed every 24 hours by cert-manager.

## What I Learned

1. Cilium's Gateway API support requires experimental CRDs (GRPCRoute, TLSRoute) — this isn't documented prominently. The error message is clear once you see it, but you have to restart the operator to see it.

2. Kubernetes RBAC `resourceNames` restrictions on `create` verbs are silently useless. The authorization check passes because there's no name to match against, but the *intent* — "only allow creating this specific named resource" — is a lie. If the thing gets deleted, you can't recreate it.

3. External-DNS with `--source=service` doesn't see Gateway resources. If you're using Cilium Gateway API, either add `--source=gateway-httproute` to External-DNS or annotate the auto-created Service. Cilium doesn't propagate annotations from Gateway to Service.

4. The cluster had been silently failing cert renewals for three days and nothing noticed because the alerting pipeline didn't exist yet. The very thing I was deploying (ntfy + PrometheusRules) would have caught this immediately. There's a metaphor in there about infrastructure bootstrapping and chickens and eggs but I'm too tired to articulate it.
