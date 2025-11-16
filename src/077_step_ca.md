# Step-CA (Again)

After implementing Cilium for networking (see [076_cilium](./076_cilium.md)), the cluster needed a proper internal Public Key Infrastructure (PKI) for managing TLS certificates. We'd set this up [before](./041_step_ca.md), so it was time to get Step-CA working with the Talos cluster.

## Why Internal PKI?

An internal PKI provides several advantages for cluster services:

- **Automated certificate issuance**: Services can request certificates declaratively
- **Short-lived certificates**: 24-hour lifetimes reduce blast radius of compromised keys
- **Centralized trust**: Single root CA for all internal services
- **GitOps-managed**: Entire PKI configuration version-controlled and automated
- **No external dependencies**: Full control over certificate issuance and revocation
- **Learning opportunity**: Deep dive into PKI, x.509, and certificate automation

For a learning cluster, building a proper PKI demonstrates production-grade security practices that are often hidden behind managed services in cloud environments.

## Architecture Overview

The PKI infrastructure consists of three layers, deployed via Flux with dependency ordering:

```
Layer 00: Foundations
  ├── cert-manager (certificate lifecycle management)
  ├── cert-manager-approver-policy (approval automation)
  ├── step-ca (Certificate Authority)
  └── step-issuer (cert-manager ↔ step-ca bridge)

Layer 01: Issuers
  ├── StepClusterIssuer (cluster-wide issuer)
  ├── CertificateRequestPolicy (approval rules)
  └── RBAC bindings (allow cert-manager to use policy)

Layer 02: Tests
  ├── Test certificate (24h lifetime)
  └── Canary certificate (2h lifetime, renews hourly)
```

Each layer depends on the previous layer being healthy before deploying, ensuring correct startup ordering.

## Component Details

### step-ca: The Certificate Authority

step-ca is Smallstep's open-source CA server. It's designed for automated certificate issuance with:

- **JWK provisioners**: Authenticate with JSON Web Keys
- **Short default lifetimes**: Encourages certificate rotation
- **REST API**: Easy integration with automation tools
- **Flexible configuration**: Claims, provisioners, and extensions

The deployment uses bootstrap mode to auto-generate a root CA and JWK provisioner on first run:

```yaml
ca:
  name: Goldentooth CA
  address: :9000
  dns: step-ca.goldentooth.net,step-ca.step-ca.svc.cluster.local
  url: https://step-ca.goldentooth.net
  db:
    enabled: true
    persistent: false  # emptyDir for homelab
  claims:
    defaultTLSCertDuration: 24h
    maxTLSCertDuration: 24h
    minTLSCertDuration: 5m

bootstrap:
  enabled: true
  configmaps: true  # Export CA cert to ConfigMap
  secrets: true     # Store CA keys in Secrets
```

The `persistent: false` setting uses emptyDir storage. While this means the certificate issuance database is lost on pod restart, the root CA itself is preserved in ConfigMaps and Secrets. Certificates simply need to be re-issued after a restart, which happens automatically thanks to cert-manager.

### cert-manager: Certificate Lifecycle Management

cert-manager is the de facto standard for Kubernetes certificate automation. It:

- Watches `Certificate` resources and creates certificate requests
- Coordinates with external CAs to sign requests
- Stores issued certificates in Kubernetes Secrets
- Automatically renews certificates before expiration

The deployment uses the official Helm chart with minimal customization:

```yaml
values:
  crds:
    enabled: true
  resources:
    requests:
      cpu: 10m
      memory: 64Mi
```

### step-issuer: The Bridge

step-issuer is a custom cert-manager Issuer that speaks step-ca's API. It translates cert-manager's `CertificateRequest` resources into step-ca API calls using JWK authentication.

The critical configuration challenge was finding the correct chart version. The step-issuer project underwent a major version jump from 0.8.x directly to 1.8.x, skipping the 0.10.x range entirely. Using `version: "1.9.x"` resolved the "chart not found" errors.

### cert-manager-approver-policy: Automated Approval

A subtle but critical component! cert-manager's built-in approver only handles internal issuers (CA, SelfSigned, Venafi). External issuers like step-issuer require explicit approval via policies.

Without approver-policy, certificate requests would sit in "pending approval" state forever:

```
NAME               APPROVED   DENIED   READY   ISSUER
test-certificate-1                     step-ca
                   ^-- stuck here
```

The approver-policy controller watches for `CertificateRequestPolicy` resources that define approval rules. But here's the catch: policies must be **bound via RBAC** to the requester!

## GitOps Structure

The deployment follows a layered GitOps approach with explicit dependencies:

```
gitops/infrastructure/pki/
├── kustomization.yaml (orchestrates all layers)
├── 00-foundations/
│   ├── flux-kustomization.yaml
│   ├── cert-manager/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── release.yaml
│   │   └── repository.yaml
│   ├── cert-manager-approver-policy/
│   │   ├── kustomization.yaml
│   │   └── release.yaml
│   ├── step-ca/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── release.yaml
│   │   ├── repository.yaml
│   │   └── service-alias.yaml
│   └── step-issuer/
│       ├── kustomization.yaml
│       └── release.yaml
├── 01-issuers/
│   ├── flux-kustomization.yaml (depends on 00-foundations)
│   ├── kustomization.yaml
│   ├── step-cluster-issuer.yaml
│   ├── certificate-request-policy.yaml
│   └── policy-rbac.yaml
└── 02-tests/
    ├── flux-kustomization.yaml (depends on 01-issuers)
    ├── kustomization.yaml
    ├── test-certificate.yaml
    └── canary-certificate.yaml
```

Each Flux Kustomization waits for the previous layer's health checks before proceeding:

```yaml
# 01-issuers/flux-kustomization.yaml
spec:
  dependsOn:
    - name: pki-foundations
  healthChecks:
    - apiVersion: certmanager.step.sm/v1beta1
      kind: StepClusterIssuer
      name: step-ca
      namespace: ""
```

## Deployment Process

The deployment was a journey through several layers of abstraction and error messages:

### 1. Initial Bootstrap Issues

Early attempts used `inject.enabled: true` to provide configuration to step-ca. This conflicted with `bootstrap.enabled: true`, causing the CA to fail initialization. The chart expects either bootstrap (auto-generate config) OR inject (provide pre-existing config), not both.

**Solution**: Use bootstrap mode exclusively, let step-ca auto-generate its CA and provisioner.

### 2. Persistence Problems

The default `persistence.enabled: true` setting tried to create a PersistentVolumeClaim. However, the correct field for disabling persistence in the step-certificates chart is `ca.db.persistent: false`, not the top-level persistence field.

**Solution**: Explicitly set `ca.db.persistent: false` to use emptyDir storage.

### 3. Service DNS Naming Mismatch

The Helm chart created a service named `step-ca-step-ca-step-certificates` (following Helm's naming pattern), but the bootstrap process generated a CA certificate with SANs for `step-ca.step-ca.svc.cluster.local`. The StepClusterIssuer tried to connect to the short name, but TLS verification failed:

```
certificate is valid for step-ca.goldentooth.net, step-ca.step-ca.svc.cluster.local,
not step-ca-step-ca-step-certificates.step-ca.svc.cluster.local
```

**Solution**: Create a Service alias with the short name that matches the certificate SAN:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: step-ca
  namespace: step-ca
spec:
  type: ClusterIP
  ports:
    - name: https
      port: 443
      targetPort: 9000
  selector:
    app.kubernetes.io/name: step-certificates
    app.kubernetes.io/instance: step-ca-step-ca
```

This provides `step-ca.step-ca.svc.cluster.local` DNS resolution while preserving the Helm-generated service name.

### 4. Certificate Request Approval Deadlock

After resolving connectivity, certificate requests were created but never approved:

```bash
$ kubectl get certificaterequest -n cert-test
NAME               APPROVED   DENIED   READY   ISSUER
test-certificate-1                     step-ca
```

The cert-manager logs showed a helpful message:

```
Request is not applicable for any policy so ignoring
```

This revealed two missing pieces:

#### Missing Component 1: cert-manager-approver-policy

The approver-policy controller wasn't installed. cert-manager's built-in approver only handles internal issuers, so external issuers like step-issuer need the policy controller.

#### Missing Component 2: RBAC Bindings

Even after creating a `CertificateRequestPolicy`, requests were still ignored! The policy controller requires RBAC bindings to allow requesters to "use" policies:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: approve-step-ca-requests:use
rules:
  - apiGroups: [policy.cert-manager.io]
    resources: [certificaterequestpolicies]
    verbs: [use]
    resourceNames: [approve-step-ca-requests]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cert-manager:approve-step-ca-requests
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: approve-step-ca-requests:use
subjects:
  - kind: ServiceAccount
    name: cert-manager-cert-manager
    namespace: cert-manager
```

This pattern follows Kubernetes' principle of explicit authorization: just because a policy exists doesn't mean anyone can use it.

### 5. Policy Rejection: Missing Organization Field

Once RBAC was in place, certificate requests were finally being evaluated—but denied!

```
No policy approved this request: [approve-step-ca-requests:
spec.allowed.subject.organizations: Invalid value: []string{"Goldentooth"}: no allowed values]
```

The test certificate included `subject.organizations: ["Goldentooth"]`, but the policy's `allowed` section didn't include an organizations field. In cert-manager-approver-policy, the `allowed` section works as a **whitelist**: any field in the certificate request must be explicitly allowed, or the request is denied.

**Solution**: Add organization to the allowed fields:

```yaml
allowed:
  subject:
    organizations:
      values:
        - "Goldentooth"
```

This security-by-default behavior prevents privilege escalation via unexpected certificate fields.

## Security Hardening

After basic functionality was working, several security improvements were implemented:

### 1. Tighten Approval Policy

The initial policy used wildcards for everything:

```yaml
# Before: Overly permissive
allowed:
  commonName: { value: "*" }
  dnsNames: { values: ["*.goldentooth.local", "*.goldentooth.net", "*.svc.cluster.local"] }
  ipAddresses: { values: ["*"] }
  organizations: { values: ["*"] }
```

This was refined to actual cluster requirements:

```yaml
# After: Restricted to actual needs
allowed:
  commonName: { value: "*" }  # OK for internal CA
  dnsNames:
    values:
      - "*.goldentooth.local"
      - "*.goldentooth.net"
      - "*.svc.cluster.local"
      - "*.*.svc.cluster.local"  # Namespaced services
  ipAddresses:
    values:
      - "10.*"  # Cluster IP range only
  subject:
    organizations:
      values:
        - "Goldentooth"  # Specific org only
```

IP addresses are limited to the internal `10.*` range (cluster IPs), excluding the home network `192.168.*` range which should never receive cluster-issued certificates.

### 2. Enforce Certificate Duration Limits

Two layers of duration enforcement provide defense-in-depth:

**Policy Layer** (primary enforcement):
```yaml
constraints:
  maxDuration: 24h
  minDuration: 5m
```

The policy rejects any certificate request outside these bounds before it reaches the CA.

**CA Layer** (backup enforcement):
```yaml
ca:
  claims:
    defaultTLSCertDuration: 24h
    maxTLSCertDuration: 24h
    minTLSCertDuration: 5m
```

While the CA claims are configured in the HelmRelease, they may not apply in bootstrap mode (the ConfigMap shows `claims: null`). However, step-ca's default claims already enforce 24h maximum, providing a reasonable baseline even without explicit configuration.

The **policy layer is the primary defense** and is cleanly expressed in GitOps. This follows the principle of enforcing security at the earliest possible point in the request flow.

## Continuous Validation with Canary Certificates

To ensure certificate renewal continues working, a canary certificate with aggressive rotation was added:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: canary-certificate
  namespace: cert-test
spec:
  secretName: canary-tls-secret
  duration: 2h
  renewBefore: 1h  # Renews when 50% lifetime remains
  commonName: canary.goldentooth.local
  dnsNames:
    - canary.goldentooth.local
  issuerRef:
    name: step-ca
    kind: StepClusterIssuer
    group: certmanager.step.sm
```

This certificate:
- Has a 2-hour lifetime
- Renews when 1 hour remains (50% threshold)
- Therefore renews approximately **every hour**

If any component of the PKI stack breaks (step-ca unavailable, policy misconfigured, StepClusterIssuer invalid), the canary will fail to renew within 1-2 hours instead of 20+ hours for a normal 24h certificate. This provides early warning of PKI issues.

The canary pattern is purely declarative—just another Certificate resource—requiring no additional infrastructure. It continuously validates the entire certificate request → approval → issuance → renewal path.

I might reduce this to e.g. 15 minutes or something, but this should be fine.

## Verification

After all components were deployed, verification confirmed the PKI was fully operational:

```bash
$ kubectl get certificate -n cert-test
NAME                 READY   SECRET              AGE
canary-certificate   True    canary-tls-secret   1h
test-certificate     True    test-tls-secret     5h
```

Both certificates show `READY`, meaning they were successfully issued, approved, and stored in Secrets.

Checking the certificate requests:

```bash
$ kubectl get certificaterequest -n cert-test
NAME                   APPROVED   DENIED   READY   ISSUER
canary-certificate-1   True                True    step-ca
test-certificate-1     True                True    step-ca
```

Both show `APPROVED=True` and `READY=True`, confirming the approval policy and RBAC bindings are working correctly.

Inspecting the canary certificate shows the short lifetime and renewal timing:

```bash
$ kubectl get secret -n cert-test canary-tls-secret -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | openssl x509 -noout -text | grep -A 2 "Validity"
        Validity
            Not Before: Nov 16 03:25:27 2025 GMT
            Not After : Nov 16 05:26:27 2025 GMT
```

The 2-hour validity period is correct (03:25 → 05:26).

Checking the renewal time:

```bash
$ kubectl describe certificate -n cert-test canary-certificate | grep "Renewal Time"
Renewal Time: 2025-11-16T04:26:27Z
```

The certificate will renew at 04:26 (1 hour after issuance), confirming the `renewBefore: 1h` configuration.

## Defense-in-Depth: Multiple Enforcement Layers

The final architecture implements three overlapping security controls:

1. **CertificateRequestPolicy constraints** (Layer 01)
   - Enforces max duration: 24h
   - Enforces min duration: 5m
   - Restricts DNS names to known patterns
   - Restricts IPs to internal cluster range
   - Restricts organization field
   - **Primary enforcement point** (cleanest GitOps expression)

2. **step-ca claims** (Layer 00)
   - Configured in HelmRelease values
   - May not apply due to bootstrap mode
   - Provides backup enforcement if requests bypass policy
   - step-ca defaults are reasonable (24h max)

3. **Certificate resource defaults** (Layer 02)
   - Applications request sensible values (24h)
   - Well-behaved clients don't try to abuse the system

This layered approach means even if one layer is misconfigured, the others provide protection. The policy layer is the most important because:

- It's **declarative and version-controlled** in Git
- It catches bad requests **before they reach the CA**
- It's **easy to audit and modify** via pull requests
- It follows the **Kubernetes admission controller pattern**

Similar to how Pod Security admission controls prevent privileged pods, CertificateRequestPolicy prevents unauthorized certificates.

## Lessons Learned

### 1. Bootstrap vs Inject Modes Are Mutually Exclusive

step-ca's Helm chart supports either auto-generating configuration (bootstrap) or injecting pre-existing configuration (inject), but not both. For a new deployment, bootstrap mode is simpler and more GitOps-friendly.

### 2. External Issuers Need Explicit Approval Infrastructure

cert-manager's built-in approver only handles cert-manager's own issuers. External issuers require:
- The approver-policy controller installed
- A CertificateRequestPolicy defining rules
- RBAC bindings allowing requesters to use the policy

This three-part requirement isn't obvious from documentation and was discovered through error messages.

### 3. Approval Policies Are Whitelists, Not Filters

Any field in a certificate request must be explicitly allowed in the policy's `allowed` section. Missing fields result in denial, not omission. This security-by-default behavior prevents privilege escalation but requires careful policy authoring.

### 4. Service Naming Matters for TLS Verification

When the Helm chart's service name doesn't match the CA's certificate SANs, TLS verification fails. Creating a service alias that matches the expected DNS name solves this without modifying chart defaults.

### 5. Canary Certificates Provide Continuous Validation

Rather than waiting for the first production certificate renewal to discover issues, a short-lived canary certificate provides hourly validation of the entire PKI stack. This is a pure GitOps pattern requiring no additional tooling.

### 6. Policy Enforcement > CA Enforcement for GitOps

While configuring step-ca's claims seems like the "right" place for enforcement, policies provide better GitOps expressiveness, earlier validation, and easier auditability. The CA provides defense-in-depth, but the policy is the primary control.
