# Step-CA: ACME Provisioner and the Inject Mode Migration

## Why ACME

The PKI stack was already working: step-ca issues certs, step-issuer bridges to cert-manager, cert-manager handles the lifecycle. Every certificate in the cluster goes through that pipeline. It's fine. It works.

But ACME is the lingua franca of certificate provisioning. Every reverse proxy, every ingress controller, every piece of software that's ever heard of Let's Encrypt speaks ACME natively. Having an ACME endpoint on the internal CA means services can request certs without needing to know anything about step-issuer or cert-manager. Standard protocol, standard clients. cert-manager itself has an ACME issuer type. It's just a more universal interface.

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

After way too many commits trying to make this work, I gave up and inlined everything. The passwords are base64-encoded in the HelmRelease, the encrypted intermediate key is right there in the YAML. Is it ideal? No. But the intermediate key is already AES-256-CBC encrypted with the CA password, so it's not like it's sitting there in plaintext.

The base64 passwords though... yeah. That's on the list.

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
- Deleted the unused SOPS secret.yaml and its kustomization reference
- Removed the SOPS decryption block from the Flux Kustomization (no encrypted files remain)

## What's Left

- **SOPS for passwords**: The base64-encoded CA and provisioner passwords are still inline in the HelmRelease. Need to find a working approach for SOPS protection that doesn't involve the broken `valuesFrom` mechanism. Maybe encrypt the entire release.yaml? Or use a different secret injection pattern.
- **ACME integration testing**: The provisioner is live, but I haven't actually tested an end-to-end ACME flow yet. Need to set up a cert-manager ACME Issuer pointing at the internal CA, or use `certbot` with the `--server` flag to hit `https://step-ca.goldentooth.net/acme/acme/directory`.
- **DNS-01 challenge support**: The ACME provisioner supports it, and we have a Route 53 connection, but the actual DNS solver configuration in cert-manager is a separate piece of work.
