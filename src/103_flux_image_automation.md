# Flux Image Automation: Push to Deploy

## The Problem With `latest`

Last entry I got Forgejo building Docker images automatically. Push to GitHub, mirror syncs, Actions builds the image, Kaniko pushes to the registry. Wonderful. Except the deployment still said `image: goldentooth-mcp:latest` and `imagePullPolicy: IfNotPresent`, which means Kubernetes had absolutely no reason to pull a new image. The deployment hadn't changed. The tag hadn't changed. As far as Flux was concerned, nothing happened.

I could have set `imagePullPolicy: Always`, but that's the Kubernetes equivalent of "close your eyes and hope." No audit trail. No rollback. No way to know which image is actually running. If the registry goes down, your next pod restart pulls... nothing.

The real fix: Flux image automation.

## The Architecture

Flux has two optional controllers that most people don't install because `flux bootstrap` doesn't include them by default:

- **image-reflector-controller** — scans container registries for new tags
- **image-automation-controller** — commits updated tags back to your git repo

Together they form a closed loop:

```
CI pushes new image tag → reflector scans registry →
policy selects newest tag → automation commits new tag to gitops →
Flux reconciles → pod runs new image
```

The key insight: the automation controller doesn't update the running cluster directly. It commits to git. The *existing* Flux reconciliation loop picks up that commit and deploys it. GitOps all the way down. Your git history is a complete record of every image that was ever deployed.

## Moving the Deployment to gitops

First problem: the MCP deployment manifests lived in the `mcp` repo, not in `gitops`. The image automation controller commits tag updates to a git repo, and having it push to `mcp` while Flux watches `gitops` would be... confusing.

Moved the manifests to `gitops/apps/mcp/` — namespace, deployment, service, kustomization. The deployment's image line gets a special marker comment:

```yaml
image: registry.goldentooth.net/goldentooth-mcp:latest # {"$imagepolicy": "flux-system:mcp"}
```

That JSON comment is how the automation controller knows *where in the file* to write the updated tag. It's a setter. Flux scans the file for these markers and replaces the value before the comment with whatever the named ImagePolicy resolved to.

## Tag Strategy: Why Not SHA?

The first CI build tagged images with the full git SHA: `d53a7b81d36805660f2cd5c6e66d42f3d6dfdb2e`. Problem: SHA hashes aren't chronologically sortable. If you have tags `abc123` and `def456`, which one is newer? You can't tell without checking the registry's metadata.

Flux's ImagePolicy needs a sortable tag format. Options:
- Semver (`1.2.3`) — overkill for a single-dev project
- Timestamp (`1710886400`) — sortable but opaque
- Run number (`2`, `3`, `4`...) — Forgejo's `github.run_number` is a monotonically increasing integer

I went with `<run_number>-<sha>`: `2-f62ba56a999ec99a7375fed529acf4b336691bdb`. The ImagePolicy extracts the numeric prefix and picks the highest:

```yaml
spec:
  policy:
    numerical:
      order: asc
  filterTags:
    pattern: '^(?P<num>[0-9]+)-[a-f0-9]+$'
    extract: '$num'
```

`filterTags` ignores `latest` (doesn't match the pattern), extracts the run number, and `numerical.order: asc` picks the highest. Clean, deterministic, always increasing.

## Installing the Controllers

Here's where I learned that "the CRDs are in gotk-components.yaml" does not mean "the controllers are running." The initial `flux bootstrap` didn't include the image controllers. Running `kubectl api-resources | grep image` returned nothing.

Fix:

```bash
flux install \
  --components-extra=image-reflector-controller,image-automation-controller \
  --export > clusters/goldentooth/flux-system/gotk-components.yaml
```

This regenerated the entire components file with the two extra controllers (+1944 lines). Committed, pushed, reconciled. Both controllers came up in seconds.

## The TLS Saga

The ImageRepository's first scan failed:

```
scan failed: tls: failed to verify certificate: x509: certificate has expired
```

Two problems stacked on top of each other:

**Problem 1: `insecure: true` doesn't mean what you think.**

In Flux ImageRepository, `insecure: true` means "use HTTP instead of HTTPS." It does NOT mean "skip TLS certificate verification." Coming from `curl -k` land, this was... not intuitive. For a registry with a private CA, you need `certSecretRef` pointing to a secret containing the CA certificate:

```yaml
spec:
  certSecretRef:
    name: registry-ca
```

I created a secret containing the Step-CA root cert (it's a public cert, not sensitive, no SOPS needed) and referenced it.

**Problem 2: The registry was serving an expired cert.**

cert-manager had renewed the TLS certificate in the Kubernetes secret. The secret had a valid cert (checked with `openssl x509 -noout -dates`). But the registry pod was still serving the *old* cert from memory. Kubernetes updates the mounted volume when a secret changes, but the Docker Registry v2 process reads the cert files at startup and never checks again.

Restarting the registry pod fixed it. But this is a ticking time bomb — every 24 hours, the cert renews, and the registry keeps serving the old one until someone notices. This needs a proper fix.

## The Deploy Key

ImageUpdateAutomation tried to push its first commit and got:

```
failed to push to remote: ERROR: The key you are authenticating with has been marked as read only
```

Of course. `flux bootstrap` creates a read-only deploy key because it only needs to *read* the repo. Image automation needs to *write* to it. GitHub doesn't support updating deploy key permissions in place, so:

```bash
# Delete the read-only key
gh api -X DELETE repos/goldentooth/gitops/keys/145157804

# Re-add with write access
gh api repos/goldentooth/gitops/keys \
  -f title="flux-system" \
  -f key="$PUBKEY" \
  -f read_only=false
```

After that, the automation controller pushed its first commit:

```
23d4bb5 chore(flux): update mcp image to registry.goldentooth.net/goldentooth-mcp:2-f62ba56a999ec99a7375fed529acf4b336691bdb
```

Fluxcdbot's first contribution. I'm unreasonably proud.

## The Result

```bash
$ kubectl get pods -n goldentooth-mcp
NAME                               READY   STATUS    RESTARTS   AGE
goldentooth-mcp-6fc48bc7d7-d88p8   1/1     Running   0          67s

$ kubectl get pods -n goldentooth-mcp -o jsonpath='{.items[0].spec.containers[0].image}'
registry.goldentooth.net/goldentooth-mcp:2-f62ba56a999ec99a7375fed529acf4b336691bdb
```

Specific image tag. Committed to git. Deployed by Flux. No human in the loop.

## The Files

```
apps/mcp/
├── namespace.yaml
├── deployment.yaml              # Has $imagepolicy setter comment
├── service.yaml
├── registry-ca-secret.yaml      # Step-CA root cert for registry TLS
├── image-repository.yaml        # Scans registry every 1m
├── image-policy.yaml            # Selects newest by run_number
├── image-update-automation.yaml # Commits tag updates to gitops
└── kustomization.yaml
```

## What's Still Broken

The registry cert reload. Every 24 hours, cert-manager renews the TLS cert, the registry keeps serving the stale one, and eventually something fails. Right now the "fix" is to restart the pod. That's not a fix, that's a prayer schedule.

Docker Registry v2 doesn't support SIGHUP-based cert reload. The cert files update on disk (Kubernetes handles that), but the process doesn't re-read them. Options include Stakater Reloader (watches secrets, restarts pods automatically), a CronJob that bounces the pod daily, or accepting that I'll get paged at 3am when the cert expires. Guess which option I'm implementing next.
