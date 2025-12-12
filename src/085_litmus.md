# Litmus: ARM64, MongoDB Hell, and Learning to Love Heterogeneous Clusters

I decided it was time to install [Litmus](https://litmuschaos.io) in the cluster. Why? Because things breaking (and the learning that follows) is the entire point of this homelab. It's not home*prod*, after all.

Litmus provides this capability with a nice UI (ChaosCenter) for designing chaos experiments and tracking their results. So I set out to install it.

What followed was a multi-hour journey through ARM64 CPU microarchitecture incompatibilities, Bitnami image repository changes, heterogeneous cluster workload placement strategies, and Kubernetes node taints. By the end, I had learned way more about MongoDB's ARM support (or lack thereof) than I ever wanted to know.

## What is Litmus?

Litmus is a CNCF chaos engineering platform for Kubernetes. It has two main deployment modes:

1. **Operator-only mode**: Just the chaos-operator and CRDs for defining chaos experiments. No UI, no persistence—just raw experiment execution via YAML.
2. **ChaosCenter mode**: Full deployment with a web UI, MongoDB for storing experiment history, and a GraphQL API server for managing experiments through a nice interface.

The ChaosCenter includes:
- **Frontend**: React web UI for designing and visualizing chaos experiments
- **GraphQL Server**: Backend API for experiment orchestration
- **Auth Server**: Authentication/authorization service
- **MongoDB**: Persistence layer for experiment definitions, execution history, and user data
- **Chaos Operator**: The actual executor that runs chaos experiments

I wanted the full ChaosCenter experience because (1) I'm here to learn new tools, and (2) a UI makes it easier to explore what chaos experiments are available and visualize their impact.

## Initial Deployment

I created the standard Flux resources in `gitops/infrastructure/litmus/`:

```yaml
# namespace.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: litmus
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/warn: baseline
```

```yaml
# release.yaml - initial attempt
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: litmuschaos
  namespace: flux-system
spec:
  interval: 24h
  url: https://litmuschaos.github.io/litmus-helm/

---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: litmus
  namespace: litmus
spec:
  interval: 30m
  chart:
    spec:
      chart: litmus
      version: ">=3.20.0 <4.0.0"
      sourceRef:
        kind: HelmRepository
        name: litmuschaos
        namespace: flux-system
      interval: 12h
  values:
    portal:
      frontend:
        service:
          type: LoadBalancer
          annotations:
            metallb.io/address-pool: default
            external-dns.alpha.kubernetes.io/hostname: chaos-center.goldentooth.net
            external-dns.alpha.kubernetes.io/ttl: "60"

    mongodb:
      enabled: true
      persistence:
        enabled: true
        size: 5Gi
```

This looked straightforward. MongoDB enabled, persistence configured, LoadBalancer service with external-dns annotation. What could go wrong?

Everything. Everything could go wrong.

## Problem 1: MongoDB Hates ARM64 (Or At Least ARM64 Hates MongoDB)

The MongoDB pod immediately started crashing with this delightful message:

```
WARNING: MongoDB requires ARMv8.2-A or higher, and your current system does not appear to implement any of the common features for that!
Illegal instruction (core dumped)
```

Okay. My Raspberry Pi 4 nodes have Cortex-A72 cores, which are **ARMv8.0-A**. They're missing the atomic instructions and other features that MongoDB 5.0+ depends on.

### Attempt 1: Try MongoDB 4.4

Fine, I thought. I'll just use an older MongoDB version that doesn't have these requirements:

```yaml
mongodb:
  image:
    tag: "4.4"
```

Nope. The image pulled was `4.4.19` or later, and **MongoDB 4.4.19+ ALSO requires ARMv8.2-A**. Same illegal instruction errors.

I needed MongoDB 4.4.18 or earlier specifically. But wait, there's more bad news...

### Attempt 2: Official Mongo Image Structure

I tried using the official `mongo:4.4.18` image:

```yaml
mongodb:
  image:
    registry: docker.io
    repository: mongo
    tag: "4.4.18"
```

This pulled fine, but then the pod failed with:

```
Attempted to create a lock file on a read-only directory: /data/db
```

Why? Because the **official mongo image uses `/data/db` for data paths**, while the **Bitnami MongoDB chart expects `/bitnami/mongodb` paths**. The volume mounts were all wrong. The official image and Bitnami chart are structurally incompatible.

### Attempt 3: Community ARM64 Bitnami-Compatible Image

I found a community-maintained ARM64 MongoDB image that's Bitnami-compatible: `dlavrenuek/bitnami-mongodb-arm:7.0`

```yaml
mongodb:
  image:
    registry: docker.io
    repository: dlavrenuek/bitnami-mongodb-arm
    tag: "7.0"
```

This looked promising! The image pulled successfully, the paths matched, and...

```
WARNING: MongoDB requires ARMv8.2-A or higher
Illegal instruction (core dumped)
```

FML. This was MongoDB 7.0, so of course it still had the ARMv8.2-A requirement. Same problem, just with more steps.

### Attempt 4: Bitnami Image Repository Changes (August 2025)

At this point I tried to go back to official Bitnami images with specific version tags:

```yaml
mongodb:
  image:
    registry: docker.io
    repository: bitnami/mongodb
    tag: "7.0"
```

This failed with `ImagePullBackOff` and "not found" errors. After some investigation, I discovered that **Bitnami changed their free tier in August 2025**. They now only offer `latest` tags for free users—version-specific tags like `7.0`, `8.0`, etc. require their paid legacy repository.

Great. Just great.

So I was stuck:
- ARM64 MongoDB images need ARMv8.2-A (which my Pi 4s don't have)
- Official mongo images are incompatible with Bitnami charts
- Bitnami free tier doesn't offer version pinning
- Community images still run into ARMv8.2-A issues

## The Plot Twist: Raspberry Pi 5 to the Rescue

Just as I was about to resign myself to running MongoDB on Velaryon (the x86 GPU node), I Googled, and much to my surprise, Pi 5s support ARMv8.2-A.

My cluster has 4 Pi 5 nodes (manderly, norcross, oakheart, payne) that can run MongoDB and its tools just fine. No need to waste the x86 node on this!

## CPU Architecture Labels

To properly select Pi 5 nodes (and avoid Pi 4 nodes), I added CPU architecture labels to all nodes in `cluster/talconfig.yaml`:

- **Pi 4 nodes** (12 nodes): `cpu.arch: armv8.0-a`
- **Pi 5 nodes** (4 nodes): `cpu.arch: armv8.2-a`
- **Velaryon** (x86): `cpu.arch: x86-64`

This makes workload requirements explicit and self-documenting. When you see `nodeSelector: cpu.arch: armv8.2-a`, you immediately understand *why* - it needs ARMv8.2-A instructions that older ARM cores don't have.

After regenerating and applying the Talos configs, I could use this label to pin MongoDB and the Litmus application pods to Pi 5 nodes.

## Problem 2: Replica Set vs Standalone

I initially deployed MongoDB in `standalone` mode, but discovered that Litmus init containers are hardcoded to check `rs.status()` - a **replica set status command**. On standalone MongoDB, this command fails because there's no replica set configured.

The init container script:
```bash
until [[ $(mongosh -u ${DB_USER} -p ${DB_PASSWORD} ${DB_SERVER} --eval 'rs.status()' | grep 'ok' | wc -l) -eq 1 ]]; do
  sleep 5;
  echo 'Waiting for the MongoDB to be ready...';
done
```

The solution: use a **single-member replica set** instead of standalone. A replica set with one member is still a valid replica set (so `rs.status()` works), but without the overhead of replication.

## Problem 3: The Helm Chart Template Mystery

Even after switching to a single-member replica set and adding `cpu.arch` labels, the pods still weren't scheduling on Pi 5 nodes. I had configured nodeSelectors in the values:

```yaml
portal:
  server:
    authServer:
      nodeSelector:
        cpu.arch: armv8.2-a  # ❌ Not used by templates!
    graphqlServer:
      nodeSelector:
        cpu.arch: armv8.2-a  # ❌ Not used by templates!
```

The HelmRelease showed these values, but the Deployment didn't have nodeSelector at all. Why?

I checked the [Helm chart templates](https://raw.githubusercontent.com/litmuschaos/litmus-helm/master/charts/litmus/templates/server-deployment.yaml) and found:

```yaml
{{- with .Values.portal.server.nodeSelector }}
```

The template looks for `portal.server.nodeSelector` at the **parent level**, not at the individual `authServer`/`graphqlServer` sublevels!

This is a discrepancy between the chart's values.yaml (which defines child-level nodeSelectors) and the templates (which only use the parent-level one). The values suggest per-component control, but the templates don't implement it.

## The Final Working Configuration

After all that, here's what actually works in `gitops/infrastructure/litmus/release.yaml`:

```yaml
portal:
  server:
    # NodeSelector at PARENT level (not child level!)
    nodeSelector:
      cpu.arch: armv8.2-a

    graphqlServer:
      resources: { ... }

    authServer:
      resources: { ... }

mongodb:
  enabled: true
  architecture: replicaset  # Not standalone!
  replicaCount: 1           # Single-member replica set

  arbiter:
    enabled: false  # No arbiter needed for single member

  nodeSelector:
    cpu.arch: armv8.2-a  # Pin to Pi 5 nodes

  image:
    registry: docker.io
    repository: bitnami/mongodb
    tag: "latest"

  persistence:
    enabled: true
    size: 5Gi
    storageClass: "local-path"

  resources:
    requests:
      memory: "512Mi"
      cpu: "250m"
    limits:
      memory: "1Gi"
      cpu: "500m"
```

MongoDB pod started successfully on a Pi 5 node! Finally!

```
NAME                     READY   STATUS    RESTARTS   AGE
litmus-mongodb-0         1/1     Running   0          2m     NODE: norcross (Pi 5)
```

## Problem 4: Local Path Provisioner and PodSecurity

The MongoDB PVC tried to provision on a Pi 5 node (good!), but failed:

```
failed to provision volume: pods "helper-pod-create-pvc-..." is forbidden:
violates PodSecurity "baseline:latest": hostPath volumes
```

The local-path-provisioner creates helper pods **in the target namespace** to set up volumes. Those helper pods need hostPath mounts, which violate `baseline` PodSecurity.

The fix: change the `litmus` namespace to `privileged` PodSecurity. This makes sense anyway - chaos experiments will likely need elevated privileges to inject failures.

After this change, the PVC provisioned successfully and MongoDB started on a Pi 5 node.
