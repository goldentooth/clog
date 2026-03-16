# Step-CA: ACME Provisioner and the Inject Mode Migration

## Why ACME

The PKI stack was already working: step-ca issues certs, step-issuer bridges to cert-manager, cert-manager handles the lifecycle. Every certificate in the cluster goes through that pipeline. It's fine. It works.

But ACME is the _lingua franca_ of certificate provisioning. Every reverse proxy, every ingress controller, every piece of software that's ever heard of Let's Encrypt speaks ACME natively. Having an ACME endpoint on the internal CA means services can request certs without needing to know anything about step-issuer or cert-manager. Standard protocol, standard clients. cert-manager itself has an ACME issuer type. It's just a more universal interface.

The goal: add an ACME provisioner alongside the existing JWK one. Don't break the existing flow. Give services a second, standards-based path to certificates.

## The Helm Chart's Two Modes

The step-certificates Helm chart has two configuration modes: **bootstrap** and **inject**. Bootstrap mode is what I'd been using — you give the chart a CA name and password, and it generates all the PKI material on first install. Quick to set up, but the CA material lives in Kubernetes Secrets created by the chart, and the `ca.json` config is generated from a limited set of Helm values.

The problem: bootstrap mode doesn't expose provisioner configuration in the Helm values. You get one JWK provisioner, and that's it. Want to add ACME? Tough luck. You'd have to `kubectl exec` into the pod and use `step ca provisioner add`, which is imperative configuration that disappears the next time the pod restarts.

**Inject mode** is the declarative alternative. You provide everything: root cert, intermediate cert, intermediate key, passwords, and the full `ca.json` config as a YAML object in the Helm values. The chart just mounts what you give it. Full control.

So: migrate from bootstrap to inject mode, and while we're at it, add the ACME provisioner to `ca.json`.

## Extracting the CA Material

First step was pulling everything out of the running cluster. The bootstrap mode had created a bunch of Secrets:

```bash
# Root and intermediate certs
kubectl get secret -n step-ca step-ca-step-ca-step-certificates-certs \
  -o jsonpath='{.data.root_ca\.crt}' | base64 -d > root_ca.crt
kubectl get secret -n step-ca step-ca-step-ca-step-certificates-certs \
  -o jsonpath='{.data.intermediate_ca\.crt}' | base64 -d > intermediate_ca.crt

# The intermediate key (already encrypted with AES-256-CBC)
kubectl get secret -n step-ca step-ca-step-ca-step-certificates-secrets \
  -o jsonpath='{.data.intermediate_ca_key}' | base64 -d > intermediate_ca_key.pem

# Passwords
kubectl get secret -n step-ca step-ca-step-ca-step-certificates-ca-password \
  -o jsonpath='{.data.password}' | base64 -d > ca_password.txt
kubectl get secret -n step-ca step-ca-step-ca-step-certificates-provisioner-password \
  -o jsonpath='{.data.password}' | base64 -d > provisioner_password.txt

# The running ca.json config
kubectl exec -n step-ca step-ca-step-ca-step-certificates-0 -- \
  cat /home/step/config/ca.json > ca.json
```

The `ca.json` had the JWK provisioner with its encrypted key, the database config, TLS settings — everything needed to reconstruct the CA configuration declaratively.

## The Config

The ACME provisioner addition to `ca.json` was straightforward. Just another entry in `authority.provisioners`:

```yaml
- type: ACME
  name: acme
  claims:
    defaultTLSCertDuration: 24h
    maxTLSCertDuration: 24h
    minTLSCertDuration: 5m
```

Same lifetime constraints as the JWK provisioner. All three challenge types (HTTP-01, TLS-ALPN-01, DNS-01) are enabled by default in step-ca, no explicit configuration needed.

I also added a CA-level policy to restrict what the CA will sign, mirroring the existing CertificateRequestPolicy:

```yaml
authority:
  policy:
    x509:
      allow:
        dns:
          - "*.goldentooth.local"
          - "*.goldentooth.net"
          - "*.svc.cluster.local"
        ip:
          - "10.0.0.0/8"
```

One gotcha here: the CertificateRequestPolicy allows `*.*.svc.cluster.local` (for `service.namespace.svc.cluster.local` names), but step-ca's policy engine only supports single-level wildcards. So namespaced service DNS names work through cert-manager (which validates before forwarding to step-ca) but not through direct ACME requests. Fine for now.

## The Migration: A Series of Unfortunate Helm Upgrades

This is where things got interesting. By which I mean nine commits and a lot of staring at Flux logs.

### Problem 1: Secret Type Immutability

Bootstrap mode creates Secrets with custom types like `smallstep.com/ca-password`. Inject mode creates them as `Opaque`. Kubernetes doesn't let you change a Secret's type. The Helm upgrade just... fails silently.

Fix: `force: true` on the HelmRelease upgrade spec. This tells Helm to delete and recreate resources instead of patching them. Nuclear option, but necessary for the one-time migration.

```yaml
upgrade:
  force: true
  remediation:
    retries: 3
```

### Problem 2: The valuesFrom Misadventure

I wanted to keep the passwords out of the HelmRelease by putting them in a SOPS-encrypted Secret and using Flux's `valuesFrom` to merge them in. Seemed clean. The Secret would hold a `values.yaml` key with the nested inject.secrets values, and Flux would merge it with the HelmRelease values.

Created the SOPS-encrypted Secret. Added `valuesFrom` to the HelmRelease. Added the SOPS decryption block to the Flux Kustomization. Pushed.

Nothing happened. The passwords were empty. The Helm values showed the inject.secrets fields as blank strings. Flux reported the reconciliation as successful.

I tried: restructuring the Secret key format, different nesting levels, different YAML structures, placeholder values in the HelmRelease for the override to replace. None of it worked. The valuesFrom merge just... didn't merge.

After way too many commits trying to make this work, I gave up and inlined everything temporarily. Moved on to get the migration working, then came back and [figured out what went wrong](#fixing-valuesfrom-the-shallow-merge-trap).

### Problem 3: The Double Wildcard

Step-ca started, loaded config, immediately exited:

```
error: authority policy: x509 policy: invalid DNS name *.*.svc.cluster.local
```

Turns out step-ca only allows single-level wildcard prefixes in its policy engine. `*.svc.cluster.local` is fine. `*.*.svc.cluster.local` is not. Removed it, step-ca started.

### Problem 4: Stale Helm Releases

After all the failed upgrades and rollbacks, there were orphaned Helm release Secrets (`sh.helm.release.v1.step-ca-step-ca.v3` through `v8`) that prevented clean upgrades. Had to manually clean those out.

## The Result

After all that:

```
step-ca-step-ca-step-certificates-0   1/1     Running   0          30s
---
2026/03/15 18:19:31 Config file: /home/step/config/ca.json
2026/03/15 18:19:31 The primary server URL is https://step-ca.goldentooth.net:9000
2026/03/15 18:19:31 Root certificates are available at https://step-ca.goldentooth.net:9000/roots.pem
2026/03/15 18:19:31 Additional configured hostnames: step-ca.step-ca.svc.cluster.local
2026/03/15 18:19:31 X.509 Root Fingerprint: c733522c1d640662cca00f19524d861c69ba4d193be5e41b28b2b9efa024b126
2026/03/15 18:19:31 Serving HTTPS on :9000 ...
```

Both provisioners confirmed alive:

```bash
wget -qO- --no-check-certificate https://127.0.0.1:9000/provisioners
```

```json
{
  "provisioners": [
    {"type": "JWK", "name": "admin", "claims": {"minTLSCertDuration": "5m0s", "maxTLSCertDuration": "24h0m0s", "defaultTLSCertDuration": "24h0m0s"}},
    {"type": "ACME", "name": "acme", "claims": {"minTLSCertDuration": "5m0s", "maxTLSCertDuration": "24h0m0s", "defaultTLSCertDuration": "24h0m0s"}}
  ]
}
```

All existing certificates still `Ready: True`. The canary certificate renewed on schedule. Nothing broke.

## Cleanup

Post-migration cleanup:
- Removed `force: true` from the HelmRelease (no longer needed)
- Passwords moved to SOPS-encrypted Secret with `valuesFrom` + `targetPath` (see below)
- Only the AES-256-CBC encrypted intermediate key remains inline in the HelmRelease

## Fixing valuesFrom: The Shallow Merge Trap

Remember how I gave up on SOPS-encrypted secrets via `valuesFrom`? Turns out I was an idiot.

Flux's `valuesFrom` merges values in order: valuesFrom entries first, then inline `spec.values` **overwrites**. The merge is shallow at the top level. My Secret contained a YAML structure under `inject.secrets`, but the inline `values` also defined `inject:` (for certificates, config, etc.). The inline `inject:` completely replaced whatever `inject:` came from the Secret. The passwords were merged in, then immediately obliterated.

The fix: `targetPath`. Instead of putting a YAML structure in the Secret and hoping it merges, you put individual values and tell Flux exactly where to inject them:

```yaml
valuesFrom:
  - kind: Secret
    name: step-ca-secrets
    valuesKey: ca_password
    targetPath: inject.secrets.ca_password
  - kind: Secret
    name: step-ca-secrets
    valuesKey: provisioner_password
    targetPath: inject.secrets.provisioner_password
```

`targetPath` injects a scalar value at an exact dot-notation path _after_ all merging. It can't be clobbered by the inline structure. The Secret has two keys, each holding one base64-encoded password, SOPS-encrypted in git. Done.

One timing wrinkle: on the first deploy, the helm-controller tried to read the Secret before the kustomize-controller had finished decrypting and applying it. The error was `key "type:str]" has no value` — it was reading the raw SOPS ciphertext. A manual `flux reconcile helmrelease` after the Secret existed fixed it, and subsequent reconciliations are fine.

## Integration Testing: The ACME Gauntlet

With the provisioner running, I wanted to prove the whole flow works: cert-manager talks ACME to step-ca, gets a challenge, solves it, gets a cert.

### The ClusterIssuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: step-ca-acme
spec:
  acme:
    server: https://step-ca.step-ca.svc.cluster.local/acme/acme/directory
    caBundle: <base64 root CA>
    privateKeySecretRef:
      name: step-ca-acme-account-key
    solvers:
      - http01:
          gatewayHTTPRoute:
            parentRefs:
              - name: goldentooth
                namespace: gateway
                kind: Gateway
```

Account registered immediately. Good sign.

### cert-manager Gateway API: A Three-Act Feature Gate Drama

The HTTP-01 solver uses Gateway API's `HTTPRoute` to route challenge traffic through the Cilium gateway. cert-manager needs a feature gate enabled for this.

**Act 1**: Set `featureGates: ExperimentalGatewayAPISupport=true` as a top-level Helm value. Passed as `--feature-gates=...` CLI flag. Controller starts, challenge fails: `gateway api is not enabled`.

**Act 2**: Moved it to the controller configuration object:

```yaml
config:
  apiVersion: controller.config.cert-manager.io/v1alpha1
  kind: ControllerConfiguration
  featureGates:
    ExperimentalGatewayAPISupport: true
```

ConfigMap updated correctly. Controller logs: `"not starting controller as it's disabled" controller="gateway-shim"`. Same error. What.

**Act 3**: Turns out cert-manager 1.15 promoted Gateway API to beta and **deprecated** the `ExperimentalGatewayAPISupport` feature gate. It's a no-op. Accepts the value, parses it fine, does absolutely nothing with it. The actual setting in 1.16 is:

```yaml
config:
  apiVersion: controller.config.cert-manager.io/v1alpha1
  kind: ControllerConfiguration
  enableGatewayAPI: true
```

A completely different field name, at a completely different level in the config structure, with zero deprecation warnings in the logs. Classic.

After that, `gateway-shim` showed up in the enabled controllers list and the solver started creating HTTPRoutes.

### DNS Propagation: The 24-Hour Brick Wall

The HTTP-01 flow works like this: cert-manager creates a solver pod, a service, and an HTTPRoute. The ACME server (step-ca) makes an HTTP request to `http://<domain>/.well-known/acme-challenge/<token>` and the solver pod responds with the proof. Simple.

Except the domain needs to resolve. For `acme-test.goldentooth.net`, external-dns saw the new HTTPRoute and created an A record in Route 53 pointing to the gateway IP. Public DNS (8.8.8.8, 1.1.1.1) had it within seconds. But inside the cluster, the domain didn't resolve.

The DNS chain: pod → CoreDNS → Talos node resolver (127.0.0.53) → router (10.4.0.1) → upstream DNS. The router had cached an NXDOMAIN response from before the record existed. The Route 53 SOA had a negative cache TTL of **86400 seconds**. Twenty-four hours. The router was going to insist this domain doesn't exist for a full day.

I reduced the SOA negative TTL to 300 seconds for the future, but the existing cached NXDOMAIN was already baked in. The fix that actually unblocked things: changing CoreDNS to forward directly to `8.8.8.8` and `1.1.1.1` instead of `/etc/resolv.conf`. This also fixed a latent bug where new CoreDNS pods would crash on startup due to a loop — `/etc/resolv.conf` on Talos points to `127.0.0.53`, which is Talos's own DNS cache, which forwards to the router. When CoreDNS restarts, the loop detection plugin sees `127.0.0.53 → CoreDNS → 127.0.0.53` and kills itself. The old pods had been running since before the loop existed and were fine. New pods: instant `CrashLoopBackOff`.

Fun times.

### The Cilium Hairpin: 503 From the Inside

With DNS working, cert-manager's self-check still got 503. Manual testing from other pods returned 200. The difference: cert-manager was scheduled on `manderly`, which is also where Cilium's envoy proxy for the gateway runs.

When a pod on the same node as the gateway's envoy hits the gateway's LoadBalancer IP (`10.4.11.1`), the traffic needs to "hairpin" — leave the pod, hit the LB VIP, route back to the envoy process on the same node. This hairpin path is broken in Cilium — envoy returns 503 instead of proxying to the backend.

From any other node, the request goes across the network normally and works fine. I deleted the cert-manager pod, it rescheduled on `norcross`, and the self-check immediately started returning 200.

This is a known class of Cilium issue with LoadBalancer service traffic originating from the LB's host node. It only affects in-cluster clients hitting the external VIP from the "wrong" node. External clients are fine.

The culprit turned out to be Cilium's socket-level load balancer (`socketLB`). With `kubeProxyReplacement: true`, Cilium hooks into the socket layer via eBPF to intercept connect() calls and translate service IPs directly — bypassing the normal packet path entirely. When a pod on manderly connected to `10.4.11.1`, the eBPF program tried to short-circuit the connection to the envoy process on the same node, but got the return path wrong. The fix:

```yaml
socketLB:
  hostNamespaceOnly: true
```

This restricts the socket-level LB trick to the host network namespace only. Pods still go through the normal datapath — VXLAN tunnel, proper NAT, envoy receives the traffic like any other packet. Host-namespace processes (kubelet, node-level services) still get the fast path. One line, problem gone.

Interestingly, most of the Cilium GitHub issues about this (#31653, #33243, #35424) focus on `bpf.masquerade` and native routing mode, neither of which apply here — the cluster runs VXLAN tunnel mode. The socket-level LB is a separate interception point that can cause the same symptom through a completely different mechanism.

### Success

After deleting the stale challenge and letting cert-manager retry on its new node:

```
NAME                    READY   SECRET                 ISSUER         STATUS
acme-test-certificate   True    acme-test-tls-secret   step-ca-acme   Certificate is up to date and has not expired
```

```
Issuer: O=Goldentooth CA, CN=Goldentooth CA Intermediate CA
Not Before: Mar 16 15:45:28 2026 GMT
Not After : Mar 17 15:46:28 2026 GMT
Subject: CN=acme-test.goldentooth.net
    DNS:acme-test.goldentooth.net
```

Issued by the Goldentooth CA Intermediate, 24-hour lifetime, via ACME. The full chain worked: cert-manager registered an ACME account, requested a certificate, created an HTTP-01 solver pod with a Gateway API HTTPRoute, external-dns created the DNS record, step-ca verified the challenge, and the certificate was issued.

All three test certificates now live in the cluster:
- `test-certificate` — JWK provisioner via step-issuer (existing)
- `canary-certificate` — JWK provisioner via step-issuer (existing)
- `acme-test-certificate` — ACME provisioner via cert-manager ClusterIssuer (new)

## What's Left

- **DNS-01 challenge support**: HTTP-01 works but has the DNS propagation dependency. DNS-01 via Route 53 would be more reliable for programmatic cert issuance, and the infrastructure (external-dns, AWS credentials) already exists.
