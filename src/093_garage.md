# Garage: S3 Storage for a Post-MinIO World

## The Death of MinIO

MinIO is dead.

Not "deprecated" dead, not "we're pivoting" dead. _Dead_ dead. The GitHub repo was archived on February 13, 2026. Read-only. The once-ubiquitous open source S3-compatible object store — the thing everybody used, the thing half the self-hosted world depended on — is gone.

The timeline is a masterclass in how to destroy community trust:

1. **May 2025**: MinIO quietly removes admin features from the web UI. Then OIDC login. Then stops publishing free Docker images. The community notices but hopes it's a phase.
2. **October 2025**: MinIO declares the open source edition "maintenance mode." No new features, no bug fixes, no security patches. All development has moved to **AIStor**, their proprietary fork. They publish a smug blog post bragging about "13,061 commits separating AIStor from unmaintained OSS." CVEs are left unfixed.
3. **February 2026**: The repo is archived. Fin.

The Reddit thread titles tell the story: "Avoid MinIO: developers introduce trojan horse update stripping community edition of most features" (1.9K upvotes). "MinIO is in maintenance mode and is no longer accepting new changes" (321 upvotes). One paying customer nailed it: _"We paid for support to encourage a great open source project and this is what comes of it."_ Someone else called it "pulling a Broadcom," which... yeah.

I'd already torn down SeaweedFS (our previous object store) days earlier. MinIO was the obvious replacement candidate. Nope.

## Enter Garage

[Garage](https://garagehq.deuxfleurs.fr/) is an S3-compatible object storage system built by [Deuxfleurs](https://deuxfleurs.fr/), a French non-profit hosting collective. It's written in Rust, which is fun, and it's designed from the ground up for exactly the kind of deployment I have: small nodes, multiple sites, unreliable networks, minimal resources.

Key properties:

- **Lightweight**: Single static binary, ~19MB Docker image. Runs happily on a Raspberry Pi.
- **Distributed**: Built-in replication with configurable factor. Nodes discover each other via Kubernetes CRDs, Consul, or static bootstrap.
- **S3-compatible**: Implements enough of the S3 API that Docker registry, Loki, backup tools, etc. all work fine.
- **Simple**: No operator, no Raft consensus, no filer abstraction. Just nodes that store data and talk to each other.
- **arm64 native**: First-class aarch64 builds. Important when your cluster is 16 Raspberry Pis.
- **AGPL-3.0**: Properly open source, backed by a non-profit. No rug-pull risk.

The architecture is pleasantly straightforward compared to SeaweedFS (masters + volume servers + filer + operator) or MinIO (erasure coding, complex quorum rules, enterprise feature gates). Garage nodes are all equal. Each node stores metadata (LMDB) and data blocks. Objects are split into blocks, replicated across nodes, and a partition-based routing table determines which nodes hold which data. That's it.

## The Deployment

Four Garage nodes, one per Pi 5, running as a StatefulSet with Longhorn PVCs. Following the same raw-manifest pattern as the Docker registry — no Helm chart involved.

### Manifests

```
gitops/infrastructure/garage/
├── kustomization.yaml
├── namespace.yaml
├── rbac.yaml           # ServiceAccount + ClusterRole for K8s discovery CRD
├── secret.yaml         # SOPS-encrypted: RPC secret, admin token, metrics token
├── configmap.yaml      # garage.toml
├── statefulset.yaml    # 4 replicas on Pi 5 nodes
└── service.yaml        # Headless (RPC) + ClusterIP (S3 API)
```

### Configuration

The `garage.toml` is mercifully short:

```toml
replication_factor = 2
consistency_mode = "consistent"

metadata_dir = "/var/lib/garage/meta"
data_dir = "/var/lib/garage/data"

db_engine = "lmdb"
compression_level = 1

[kubernetes_discovery]
namespace = "garage"
service_name = "garage"
skip_crd = false

[s3_api]
api_bind_addr = "[::]:3900"
s3_region = "garage"

[admin]
api_bind_addr = "[::]:3903"
```

Kubernetes discovery means Garage manages its own CRD (`garagenodes.deuxfleurs.fr`) to find peers. No bootstrap peers to configure, no Consul dependency. The RBAC gives it permission to create and manage the CRD, and nodes find each other automatically.

Secrets (RPC secret, admin token, metrics token) are injected via environment variables from a SOPS-encrypted Secret. Garage reads `GARAGE_RPC_SECRET`, `GARAGE_ADMIN_TOKEN`, and `GARAGE_METRICS_TOKEN` from the environment, which is cleaner than templating them into the TOML.

### StatefulSet

The StatefulSet runs 4 replicas pinned to Pi 5 nodes (`node.kubernetes.io/disk-type: nvme`) with Parallel pod management. Each pod gets:

- **1Gi meta PVC** (Longhorn): LMDB metadata database
- **100Gi data PVC** (Longhorn): Actual object data blocks

So Garage sits _on top of_ Longhorn, which sits on top of the physical NVMe SSDs. Longhorn handles the block replication at the storage layer, Garage handles object replication at the S3 layer. It's replication all the way down, which is maybe excessive for a single-site cluster, but at least nothing's getting lost.

### Deployment

Pushed the manifests, Flux reconciled, all 4 pods came up Running within seconds. The nodes found each other immediately via Kubernetes discovery:

```
==== HEALTHY NODES ====
ID                Hostname  Address            Tags  Zone  Capacity  DataAvail
3aba977fad6d667b  garage-3  10.244.1.77:3901   []    dc1   100.0 GB  105.1 GB
62b886fd3a97a860  garage-2  10.244.3.193:3901  []    dc1   100.0 GB  105.1 GB
a3357ec9deb02542  garage-1  10.244.2.67:3901   []    dc1   100.0 GB  105.1 GB
a74d59c28f665708  garage-0  10.244.0.32:3901   []    dc1   100.0 GB  105.1 GB
```

But the pods weren't Ready — health checks returned 503. This is because Garage requires a cluster layout to be configured before it considers itself healthy. Fair enough.

### Cluster Layout

The layout assigns capacity and zone information to each node. Since we're single-site:

```bash
garage layout assign -z dc1 -c 100G a74d
garage layout assign -z dc1 -c 100G a335
garage layout assign -z dc1 -c 100G 62b8
garage layout assign -z dc1 -c 100G 3aba
garage layout apply --version 1
```

Output:

```
Partitions are replicated 2 times on at least 1 distinct zones.

Optimal partition size:                     781.2 MB
Usable capacity / total cluster capacity:   400.0 GB / 400.0 GB (100.0 %)
Effective capacity (replication factor 2):  200.0 GB
```

200GB effective capacity with replication factor 2. Once the layout was applied, health checks immediately started passing and all 4 pods went Ready.

## Proof of Concept: Docker Registry on Garage

The whole reason for deploying Garage was to give the Docker registry a proper S3 backend again. It had been on a filesystem PVC on Longhorn since the SeaweedFS teardown — functional but not what we want long-term.

### Bucket and Key Setup

```bash
garage bucket create docker-registry
garage key create docker-registry-key
garage bucket allow --read --write docker-registry --key docker-registry-key
```

### Registry Configuration Changes

Switched the Docker registry config from filesystem to S3:

```yaml
# Before (filesystem on Longhorn PVC)
storage:
  filesystem:
    rootdirectory: /var/lib/registry

# After (Garage S3)
storage:
  s3:
    region: garage
    bucket: docker-registry
    regionendpoint: http://garage-s3.garage.svc.cluster.local:3900
```

Updated the Deployment to inject `REGISTRY_STORAGE_S3_ACCESSKEY` and `REGISTRY_STORAGE_S3_SECRETKEY` from a SOPS-encrypted Secret, removed the PVC volume mount, and swapped `pvc.yaml` for `registry-s3-secret.yaml` in the kustomization.

Pushed, Flux reconciled, the registry came up Running immediately. Checked the logs — no S3 errors, health checks passing, TLS working.

### End-to-End Test

Pushed `alpine:latest` (all architectures) into the registry using `crane`:

```bash
crane copy alpine:latest registry.goldentooth.net/test/alpine:latest --insecure
```

All blobs pushed, all manifests stored:

```
$ curl -sk https://registry.goldentooth.net/v2/_catalog
{"repositories":["test/alpine"]}

$ curl -sk https://registry.goldentooth.net/v2/test/alpine/tags/list
{"name":"test/alpine","tags":["latest"]}
```

Pulled the manifest back — full OCI image index with amd64, arm64, armv6, armv7, i386, ppc64le, riscv64, s390x variants. Everything round-tripped cleanly through Garage.

The bucket ended up with 114 objects, 29 MiB. Not bad for a multi-arch alpine image with attestation manifests.

### PVC Cleanup

The old 50Gi Longhorn PVC (`registry-data`) was automatically cleaned up by Flux when we removed `pvc.yaml` from the kustomization. One less thing to worry about.

## Current State

The bramble now has a proper S3 object store again:

- **Garage v2.2.0**: 4 nodes on Pi 5s, 200GB effective capacity, replication factor 2
- **Docker registry**: Running on Garage S3 backend, push/pull verified
- **Storage stack**: NVMe SSDs → Longhorn (block) → Garage (object) → Applications

Next up: probably migrating Loki's storage to Garage as well. And maybe writing a more scathing obituary for MinIO.
