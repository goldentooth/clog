# Forgejo: CI/CD That Doesn't Phone Home

I've been building Docker images for the MCP server by hand. Like, on my laptop. `docker build`, `docker push`, pray the registry doesn't reject it because I forgot to trust the CA this time. It works. It's also embarrassing.

The cluster runs Flux for GitOps, has a private Docker registry, has NATS for messaging, has a whole observability stack — but the actual *build step* is me, in a terminal, like some sort of artisanal software craftsman. Time to fix that.

## The Plan

Mirror the MCP repo from GitHub into a self-hosted Forgejo instance on the cluster. Forgejo has built-in Actions (GitHub Actions-compatible CI), so when a push lands on `main`, it triggers a workflow that builds the Docker image and pushes it to the registry. No external CI service, no GitHub Actions minutes, no secrets leaving the network.

The architecture:

```
GitHub push → Forgejo mirror (≤5min) → Actions workflow →
Kaniko build → registry.goldentooth.net → Flux deploys
```

Kaniko is the key piece — it builds Docker images without needing a Docker daemon or privileged containers. Well, sort of. More on that later.

## The Deployment

### Helm Chart

Standard Flux structure: namespace, HelmRepository (OCI), HelmRelease.

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: forgejo
  namespace: flux-system
spec:
  interval: 24h
  type: oci
  url: oci://code.forgejo.org/forgejo-helm
```

The Forgejo chart is distributed as an OCI artifact, not a traditional Helm repo. Flux handles this fine with `type: oci`.

Key HelmRelease values:

```yaml
gitea:
  admin:
    existingSecret: forgejo-admin
  config:
    actions:
      ENABLED: true
    mirror:
      ENABLED: true
      MIN_INTERVAL: 5m
    server:
      DOMAIN: git.goldentooth.net
      ROOT_URL: https://git.goldentooth.net/
    service:
      DISABLE_REGISTRATION: true
    database:
      DB_TYPE: sqlite3

persistence:
  enabled: true
  storageClass: seaweedfs
  size: 10Gi

nodeSelector:
  node.kubernetes.io/disk-type: nvme
```

SQLite because I don't need Postgres for a single-user forge that mirrors one repo. SeaweedFS for persistence because it's what we've got. NVMe nodes (the Pi 5s) because they have the storage and the CPU for builds.

Actions are enabled at the server level, but you *also* have to enable them per-repository. I learned this the hard way after spending twenty minutes wondering why mirror syncs weren't triggering workflows. `has_actions: false` was the default on the mirrored repo. Cool. Thanks for that.

### The Service Name Problem

The Helm chart creates a service called `forgejo-forgejo-http`. Not `forgejo-http`, which is what you'd expect. The chart names it `<release>-<chart>-http`, and since the release name is `forgejo` and the chart name is `forgejo`, you get the stutter. My HTTPRoute initially pointed at `forgejo-http` and got a 500 from Envoy. Always `kubectl get svc` after a Helm deploy.

### Gateway Route

Same pattern as everything else — HTTPRoute in the service namespace referencing the shared gateway:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: forgejo
  namespace: forgejo
spec:
  parentRefs:
    - name: goldentooth
      namespace: gateway
      sectionName: https
  hostnames:
    - git.goldentooth.net
  rules:
    - backendRefs:
        - name: forgejo-forgejo-http
          port: 3000
```

Plus `git.goldentooth.net` in the gateway TLS certificate dnsNames. Step-CA issues a new cert within minutes.

## The Mirror

Creating the mirror is an API call:

```bash
curl -k -X POST https://git.goldentooth.net/api/v1/repos/migrate \
  -H "Content-Type: application/json" \
  -u "forgejo_admin:<password>" \
  -d '{
    "clone_addr": "https://github.com/goldentooth/mcp.git",
    "repo_name": "mcp",
    "repo_owner": "forgejo_admin",
    "service": "github",
    "mirror": true,
    "mirror_interval": "5m"
  }'
```

Five-minute mirror interval. Forgejo pulls from GitHub, not the other way around. No webhooks, no GitHub tokens, no inbound network access needed.

One gotcha: mirror syncs of *already-seen commits* don't generate push events. So if you enable Actions after the initial mirror sync, you need to push a new commit to GitHub to trigger the first workflow run. I pushed a one-line change, waited for the mirror, and the run kicked off.

## The Runner: A Comedy of Errors

The Forgejo runner is where this got interesting.

### Attempt 1: Environment Variables

The plan assumed the runner image would read `FORGEJO_RUNNER_*` environment variables and Just Work. Nope. The container's default entrypoint prints a help message and exits. The runner needs two explicit commands:

1. `create-runner-file` — generates a `.runner` config file using a shared secret
2. `daemon` — actually runs the runner

The shared secret is pre-registered in Forgejo via `forgejo-cli actions register --secret <hex>`, then the runner uses the same secret to authenticate. No OAuth dance, no token exchange. Both sides just agree on a secret ahead of time.

I split this into an init container for registration and the main container for the daemon:

```yaml
initContainers:
  - name: register
    command:
      - forgejo-runner
      - create-runner-file
      - --connect
      - --instance
      - http://forgejo-forgejo-http.forgejo.svc.cluster.local:3000
      - --name
      - bramble-runner
      - --secret
      - $(FORGEJO_RUNNER_SECRET)
```

Note: `http://`, not `https://`. The gateway terminates TLS. In-cluster traffic is plain HTTP on port 3000. Using `https` here gives you a TLS handshake error and a valuable lesson about knowing which side of the gateway you're on.

### Attempt 2: No Docker

Runner starts, connects to Forgejo, declares itself with labels... then dies:

```
Error: daemon Docker Engine socket not found
```

The runner label `ubuntu-latest:docker://node:20-bookworm` tells it to run jobs inside Docker containers. But there's no Docker daemon in the pod. The Kubernetes nodes use containerd, and the runner can't just reach in and use it.

### Attempt 3: DinD Sidecar

Solution: Docker-in-Docker as a sidecar container. The `docker:27-dind` image runs a full Docker daemon inside the pod, and the runner connects to it via a shared `/var/run/docker.sock`:

```yaml
containers:
  - name: runner
    # ...
    volumeMounts:
      - name: docker-sock
        mountPath: /var/run
  - name: dind
    image: docker:27-dind
    securityContext:
      privileged: true
    env:
      - name: DOCKER_TLS_CERTDIR
        value: ""
    volumeMounts:
      - name: docker-sock
        mountPath: /var/run
volumes:
  - name: docker-sock
    emptyDir: {}
```

`privileged: true` is the price of admission. DinD needs full kernel access to run a Docker daemon inside a container. This required setting the `forgejo` namespace to `pod-security.kubernetes.io/enforce: privileged`. I'm not thrilled about it, but it's scoped to one namespace with one deployment.

### Attempt 4: Permission Denied

Runner starts, DinD starts, they share the socket... and the runner can't connect because the socket is owned by root and the runner image runs as a non-root user. `securityContext.runAsUser: 0` fixes it. We're already in privileged territory, so running as root is the least of our concerns.

### Attempt 5: Race Condition

Runner starts faster than DinD. Docker daemon takes about 8 seconds to boot. Runner checks for the socket immediately, finds nothing, dies.

Fix: a shell wrapper that waits for the socket:

```yaml
command:
  - sh
  - -c
  - |
    while [ ! -S /var/run/docker.sock ]; do sleep 1; done
    exec forgejo-runner daemon --config /etc/runner/config.yaml
```

### Attempt 6: Labels via Config File

The runner's labels aren't set via environment variables or command-line flags. They come from a config file under `runner.labels`. Without a config file, the runner registers with no labels and can't pick up any jobs. The config lives in a ConfigMap mounted at `/etc/runner/config.yaml`:

```yaml
runner:
  file: .runner
  capacity: 1
  timeout: 3h
  labels:
    - "ubuntu-latest:docker://node:20-bookworm"
container:
  docker_host: unix:///var/run/docker.sock
```

### It Works

After six iterations, the runner connected, registered with the `ubuntu-latest` label, and started polling for jobs. The pod looks like this:

```
forgejo-runner-7fb4f856f7-g69r6    2/2     Running   0
```

Two containers: `runner` and `dind`. Zero restarts. I may have pumped my fist.

## The CI Workflow

The workflow is straightforward — checkout the code, build with Kaniko:

```yaml
name: CI Build

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        uses: docker://gcr.io/kaniko-project/executor:latest
        with:
          args: >-
            --dockerfile=Dockerfile
            --context=.
            --destination=registry.goldentooth.net/goldentooth-mcp:${{ github.sha }}
            --destination=registry.goldentooth.net/goldentooth-mcp:latest
            --skip-tls-verify
```

`--skip-tls-verify` because the registry uses our private Step-CA. Kaniko doesn't know about our CA trust chain, and teaching it would mean building a custom Kaniko image or mounting the CA cert. Skip-TLS is fine for an internal registry that's already behind the cluster network.

The first build — a Rust project, cold cache, ARM64 — took about eight minutes. Not fast. But it works, it's automatic, and I never have to think about it again.

## The Result

```bash
$ curl -ks https://registry.goldentooth.net/v2/goldentooth-mcp/tags/list
{"name":"goldentooth-mcp","tags":["d53a7b81d368...","latest"]}
```

Both tags present: `latest` and the SHA tag. The full pipeline works:

1. Push to GitHub
2. Forgejo mirrors within 5 minutes
3. Actions workflow triggers
4. Kaniko builds the Docker image inside a DinD-equipped runner pod
5. Image pushed to the private registry
6. Flux can deploy the new image

## The Files

```
infrastructure/forgejo/
├── namespace.yaml            # Namespace with privileged PodSecurity
├── admin-secret.yaml         # SOPS-encrypted admin credentials
├── repository.yaml           # OCI HelmRepository
├── release.yaml              # HelmRelease with all config
├── runner-config.yaml        # ConfigMap for runner daemon config
├── runner-secret.yaml        # SOPS-encrypted runner registration token
├── runner-deployment.yaml    # Runner + DinD sidecar
└── kustomization.yaml        # Ties it all together
```

Plus `infrastructure/gateway/routes/forgejo.yaml` for the HTTPRoute and `git.goldentooth.net` added to the gateway TLS certificate.

## What I Learned

The runner was by far the hardest part. The plan assumed env vars would work and didn't account for DinD. Six iterations to get a working runner pod. Turns out "run a CI runner inside Kubernetes" is not as simple as "deploy a container," because the runner itself needs to *run containers*, and that's a fundamentally awkward thing to do inside a container.

The plan also missed: service naming (Helm prefix stutter), in-cluster HTTP vs HTTPS, per-repo Actions enablement, and the fact that mirror syncs of existing commits don't trigger workflows. Every one of these was a 5-minute fix, but each one required discovering the problem first.

Still: the cluster now has a self-hosted CI/CD pipeline that builds Docker images from GitHub mirrors without any external dependencies. Push to GitHub, wait a few minutes, image appears in the registry. That's the dream.

Next up: maybe have Flux auto-deploy the new image. Right now it's tagged `latest` and the deployment uses `latest`, so it technically works, but `imagePullPolicy: Always` is not what you'd call a "deployment strategy."
