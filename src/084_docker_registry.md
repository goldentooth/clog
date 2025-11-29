# Docker Registry: Because Harbor is Too Good for ARM64

I needed a container registry for the cluster. Something to push my own images to, especially for KubeVirt VM disk images. Harbor seemed like the obvious choice – it's the industry standard, has a nice UI, vulnerability scanning, the works.

Except Harbor doesn't officially support ARM64 yet. There are community builds and some PRs in flight for v2.14, but I'm running a Raspberry Pi cluster. I don't have time to debug multi-arch manifest issues for a registry that's basically 7+ components just to store some container images.

So: Docker Registry v2. The reference implementation. One container. Works on ARM64. Done.

## What is Docker Registry v2?

Docker Registry v2 is the OCI Distribution spec reference implementation. It's what Harbor uses internally for the actual registry component. Everything else Harbor provides (web UI, RBAC, scanning, replication) is management layer on top.

For a single-user homelab, I don't need any of that. I just need:
- Push images (`docker push registry.goldentooth.net/myapp:v1`)
- Pull images (`docker pull registry.goldentooth.net/myapp:v1`)
- Store everything in SeaweedFS S3
- TLS via cert-manager

Registry v2 does all of this in ~50MB of RAM.

## The Architecture

```
Client (docker/skopeo)
    ↓ HTTPS (TLS via cert-manager)
LoadBalancer Service (10.4.11.8)
    ↓
Registry Pod (single container)
    ├── Config: /etc/docker/registry/config.yml
    ├── Certs: /certs/tls.{crt,key} (from cert-manager)
    └── S3 Backend → SeaweedFS Filer
                         ↓
              Volume Servers (USB SSDs)
```

The registry is stateless – all image data lives in SeaweedFS S3 (`harbor-registry` bucket), and the registry just coordinates uploads/downloads.

## The Deployment

I created the Flux structure under `gitops/infrastructure/docker-registry/`:

### Namespace and Certificate

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: docker-registry
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: registry-tls
  namespace: docker-registry
spec:
  secretName: registry-tls
  duration: 24h
  renewBefore: 8h
  commonName: registry.goldentooth.net
  dnsNames:
    - registry.goldentooth.net
  issuerRef:
    name: step-ca
    kind: StepClusterIssuer
    group: certmanager.step.sm
```

cert-manager + Step-CA handles TLS automatically with 24-hour certificate rotation.

### Registry Configuration

The registry uses a YAML config mounted from a ConfigMap:

```yaml
storage:
  redirect:
    disable: true  # Important! Explained later.
  s3:
    region: us-east-1
    regionendpoint: http://goldentooth-storage-filer.seaweedfs.svc.cluster.local:8333
    bucket: harbor-registry
    secure: false
    v4auth: true
  delete:
    enabled: true
  cache:
    blobdescriptor: inmemory
http:
  addr: :5000
  tls:
    certificate: /certs/tls.crt
    key: /certs/tls.key
```

S3 credentials come from environment variables (`REGISTRY_STORAGE_S3_ACCESSKEY` / `REGISTRY_STORAGE_S3_SECRETKEY`) loaded from a SOPS-encrypted Secret.

### Deployment and Service

Single-pod Deployment with aggressive resource limits (0.25 CPU, 256MB RAM – this is a homelab):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: docker-registry
  namespace: docker-registry
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: registry
        image: registry:2
        env:
        - name: REGISTRY_STORAGE_S3_ACCESSKEY
          valueFrom:
            secretKeyRef:
              name: registry-s3-secret
              key: accesskey
        - name: REGISTRY_STORAGE_S3_SECRETKEY
          valueFrom:
            secretKeyRef:
              name: registry-s3-secret
              key: secretkey
        volumeMounts:
        - name: config
          mountPath: /etc/docker/registry
        - name: certs
          mountPath: /certs
```

LoadBalancer Service with external-dns annotation:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: docker-registry
  annotations:
    external-dns.alpha.kubernetes.io/hostname: registry.goldentooth.net
spec:
  type: LoadBalancer
  ports:
    - port: 443
      targetPort: 5000
```

MetalLB assigns an IP, external-dns creates the DNS record, cert-manager provides the cert. Everything automatic.

## The Problems: A TLS Odyssey

I deployed everything. Flux reconciled. The registry pod started. The LoadBalancer got an IP. DNS resolved. The certificate was issued.

Time to test:

```bash
$ docker push registry.goldentooth.net/test/busybox:latest
denied:
```

### Problem 1: Certificate Chain Issues

The error was actually a TLS verification failure, but Docker just said "denied" with no details. Helpful.

The issue: Docker wasn't trusting the Step-CA certificate. I added the root CA to the macOS system keychain:

```bash
$ kubectl get configmap -n step-ca step-ca-step-ca-step-certificates-certs \
    -o jsonpath='{.data.root_ca\.crt}' > /tmp/goldentooth-ca.crt
$ sudo security add-trusted-cert -d -r trustRoot \
    -k /Library/Keychains/System.keychain /tmp/goldentooth-ca.crt
```

Docker still failed. Turns out Docker Desktop on macOS doesn't use the system keychain – it has its own certificate store at `~/.docker/certs.d/<hostname>/ca.crt`.

I copied the certificate there:

```bash
$ mkdir -p ~/.docker/certs.d/registry.goldentooth.net
$ cp /tmp/goldentooth-ca.crt ~/.docker/certs.d/registry.goldentooth.net/ca.crt
```

Restarted Docker Desktop. Still failed.

Turns out the registry was serving a certificate signed by an *intermediate* CA, not the root. I needed the full chain:

```bash
$ openssl s_client -connect registry.goldentooth.net:443 -showcerts 2>/dev/null </dev/null | \
    sed -n '/BEGIN CERTIFICATE/,/END CERTIFICATE/p' > \
    ~/.docker/certs.d/registry.goldentooth.net/ca.crt
```

Still failed.

### Problem 2: The Docker Desktop Bug

After a few more minutes of TLS debugging, I found the real issue: **Docker Desktop 4.32+ (including my version 28.0.1) has a broken `insecure-registries` implementation on macOS**.

There's a confirmed bug where the setting is completely ignored. This started in mid-2024 and affects all recent versions. Docker uploads blobs successfully, then refuses to push the manifest with "denied" – but the registry logs show Docker never even *tried* to push the manifest.

The registry was working fine. Docker blobs were uploading to S3. Docker was just... giving up before pushing the final manifest for no clear reason.

### Solution: Use Skopeo

Skopeo is a Docker alternative that doesn't require a daemon and doesn't have Docker Desktop's bugs. Installed it:

```bash
$ brew install skopeo
```

Pushed the image:

```bash
$ skopeo copy --dest-tls-verify=false \
    docker-daemon:alpine:latest \
    docker://registry.goldentooth.net/test/alpine:v1
Getting image source signatures
Copying blob sha256:0e64f2360a44...
Copying config sha256:171e65262c80...
Writing manifest to image destination
```

**It worked.** First try. Blobs, config, and manifest all pushed successfully.

Verified with Docker pull:

```bash
$ docker pull registry.goldentooth.net/test/alpine:v1
v1: Pulling from test/alpine
Digest: sha256:6ecfe31476d1...
Status: Downloaded newer image
```

Perfect. The registry works. Docker Desktop is just broken.

## Problem 3: The S3 Redirect Issue

I tested pulling with skopeo:

```bash
$ skopeo copy docker://registry.goldentooth.net/test/alpine:v1 docker-daemon:test:latest
time="2025-11-29T14:04:05-05:00" level=fatal msg="reading blob: Get \"http://goldentooth-storage-filer.seaweedfs.svc.cluster.local:8333/...\": no such host"
```

The registry was sending skopeo a 307 redirect to the *internal* SeaweedFS S3 endpoint (`.svc.cluster.local`), which isn't resolvable from outside the cluster.

### What's Happening

By default, Docker Registry sends HTTP 307 redirects for blob downloads. Instead of proxying the image layer data through itself, it tells clients: "go fetch this blob directly from S3 at this URL."

This is efficient for large registries (saves bandwidth on the registry pod), but only works if clients can reach the S3 endpoint. My S3 endpoint was cluster-internal.

### The Fix: Disable Redirects

Added one line to the registry config:

```yaml
storage:
  redirect:
    disable: true  # Registry proxies all blob data
  s3:
    # ... rest of config
```

This makes the registry proxy all blob downloads instead of redirecting clients to S3. Uses more bandwidth on the registry pod, but for a homelab with minimal usage, it's fine.

Applied the change:

```bash
$ kubectl apply -f config.yaml
$ kubectl rollout restart deployment -n docker-registry docker-registry
```

Tested again:

```bash
$ skopeo copy docker://registry.goldentooth.net/test/alpine:v1 docker-daemon:test:latest
Getting image source signatures
Copying blob sha256:5096682701dd...
Copying config sha256:171e65262c80...
Writing manifest to image destination
```

Success. Skopeo can now both push and pull.
