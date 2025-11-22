# Tekton Operator: A Journey Through CRD Management Hell

I decided to set up Tekton for CI/CD pipelines. The immediate use case is building KubeVirt VM images, but Tekton's a general-purpose pipeline system that could handle all sorts of infrastructure automation. Maybe eventually I'll run a local Git server like Forgejo and have proper push-triggered builds, but for now I just want the pipeline infrastructure in place.

## The Operator Approach

Tekton consists of multiple components: Pipelines (the core), Triggers (event handling), Dashboard (UI), CLI, etc. You can install each component separately, but Tekton recommends using their Operator for production setups. The operator provides a unified management plane - you install the operator once, then create a `TektonConfig` custom resource that declares what components you want, and the operator handles installation and lifecycle management.

This fits perfectly with the GitOps model: the operator is Layer 1, the TektonConfig is Layer 2, and actual pipeline definitions are Layer 3+.

## Initial Setup: Following the Pattern

I set up the standard Flux structure I've been using for operators:

```
infrastructure/tekton/
├── operator/
│   ├── repository.yaml         # GitRepository pointing to tektoncd/operator
│   ├── release.yaml            # HelmRelease
│   ├── kustomization.yaml      # Kustomize wrapper
│   └── flux-kustomization.yaml # Flux Kustomization
└── kustomization.yaml          # Top-level includes
```

The GitRepository points to `https://github.com/tektoncd/operator` at tag `v0.77.0`. The chart exists in the repo at `./charts/tekton-operator` but isn't published to a Helm repository yet, so I use a GitRepository as the source for the HelmRelease - Flux supports this natively.

## The CRD Question: Skip or Create?

Initially, I set `install.crds: Skip` in the HelmRelease, following the "best practice" of separating CRD lifecycle from operator lifecycle. The theory is: if CRDs are tied to a Helm release and you uninstall it, the CRDs get deleted, which cascades to deleting all custom resources (via Kubernetes garbage collection). For a CI/CD system with potentially hundreds of user-created Pipelines and TaskRuns, this would be catastrophic.

So the "proper" approach is:
- Layer 0: Install CRDs separately
- Layer 1: Install operator (depends on Layer 0)
- Layer 2: Create TektonConfig (depends on Layer 1)

But this adds complexity - another layer to manage, another dependency chain.

## First Crash: Missing CRDs

Pushed the config, Flux reconciled, and... the webhook pod immediately crashed:

```
{"level":"fatal","msg":"error deleting webhook installerset",
 "error":"the server could not find the requested resource (get tektoninstallersets.operator.tekton.dev)"}
```

The operator's webhook needs `TektonInstallerSet` CRD to exist at startup. These are the operator's *management* CRDs (TektonConfig, TektonPipeline, TektonInstallerSet, etc.) - the resources you use to *tell* the operator what to install. They're different from the *workload* CRDs (Task, Pipeline, TaskRun, etc.) that you use to actually run pipelines.

The operator can't function without its management CRDs. They're not optional.

## Attempt 1: Change to `crds: Create`

Found Tekton's documentation:
> The Tekton operator components (especially the webhook) require the CRDs to be present during startup. If you set installCRDs=false, you MUST install the CRDs manually BEFORE installing the operator.

In a GitOps environment where all TektonConfigs and Pipelines are declared in Git, is the separate CRD management really necessary? If I uninstall and the CRDs get nuked, Flux will just recreate everything from Git.

For MetalLB, Cilium, etc., I use `crds: Create` or `crds: CreateReplace` without issues because all the custom resources (IPAddressPools, CiliumNetworkPolicies) are in Git. Same logic should apply here.

Changed to `crds: Create`, pushed, reconciled. Deleted the crashing webhook pod to force a fresh start.

Still crashed with the same error. WTF?

## Attempt 2: Full Reconciliation

Maybe the CRDs were installed but the old pod was still stuck? Forced a full HelmRelease reconciliation:

```bash
flux reconcile helmrelease -n flux-system tekton-operator --force
```

Webhook pod restarted. Still crashed. Same error.

Checked if CRDs exist:
```bash
kubectl get crds | grep tekton
```

Nothing. No Tekton CRDs at all, despite `crds: Create`.

## The Real Problem: Two CRD Installation Mechanisms

Turns out there are TWO different ways to install CRDs with Helm:

**1. Helm's built-in CRD directory**
- Charts can have a `crds/` directory with CRD YAML files
- Helm installs these when you install the chart
- Flux's `install.crds: Create` controls this behavior
- This is the "standard" Helm approach

**2. Chart-specific value flags**
- Some charts template CRD resources like any other resource
- They use a value flag (like `installCRDs: true`) to control whether CRD templates are rendered
- This gives the chart more control but doesn't use Helm's standard mechanism

Checked the Tekton operator chart structure:
```
charts/tekton-operator/
├── Chart.yaml
├── values.yaml
├── templates/
└── .helmignore
```

No `crds/` directory! So `install.crds: Create` does absolutely nothing - there are no CRDs for Helm to install via its built-in mechanism.

Checked `values.yaml`:
```yaml
## If the Tekton-operator CRDs should automatically be installed and upgraded
## Setting this to true will cause a cascade deletion of all Tekton resources when you uninstall
installCRDs: false
```

There it is. The chart has `installCRDs` as a *template value* that controls whether CRD resources are generated in the templates. Defaults to false.

## The Fix

Added to the HelmRelease values:

```yaml
values:
  installCRDs: true
  resources:
    limits:
      cpu: 200m
      memory: 256Mi
    requests:
      cpu: 100m
      memory: 128Mi
```

Pushed, reconciled. Checked CRDs:

```bash
kubectl get crds | grep tekton
```

```
manualapprovalgates.operator.tekton.dev
tektonchains.operator.tekton.dev
tektonconfigs.operator.tekton.dev
tektondashboards.operator.tekton.dev
tektonhubs.operator.tekton.dev
tektoninstallersets.operator.tekton.dev
tektonpipelines.operator.tekton.dev
tektonpruners.operator.tekton.dev
tektonresults.operator.tekton.dev
tektontriggers.operator.tekton.dev
```

There they are! Webhook pod restarted and came up clean:

```bash
kubectl get pods -n tekton-operator
```

```
NAME                                                       READY   STATUS    RESTARTS      AGE
tekton-operator-tekton-operator-79df9897cd-7mf2f           2/2     Running   0             16m
tekton-operator-tekton-operator-webhook-5c455997df-2qvzp   1/1     Running   7 (11m ago)   16m
```

Both pods happy. Operator ready.

## About That Cascade Deletion Warning

The chart's values.yaml has a scary warning:
> Setting this to true will cause a cascade deletion of all Tekton resources when you uninstall the chart - danger!

This is true, but in a GitOps environment it's less catastrophic than it sounds. If I uninstall the operator:

1. Helm deletes the operator Deployment
2. Helm deletes the CRDs (because `installCRDs: true` means the chart owns them)
3. Kubernetes garbage-collects all CRs (TektonConfig, etc.)
4. The operator (now deleted) would have deleted Pipelines/Tasks, but it's gone
5. Flux sees the TektonConfig is missing and recreates it from Git
6. Wait, the CRDs are gone, so the TektonConfig can't be created
7. Chicken-egg problem during recovery

So there IS a risk during operator reinstallation. But:
- I'm not planning to uninstall the operator regularly
- If I do need to reinstall, I can just wait for the operator to come back up, then Flux recreates everything
- The alternative (Layer 0 CRD management) adds ongoing complexity for every upgrade

For this cluster's scale and use case, the tradeoff is worth it. If I were running a multi-tenant CI/CD platform with hundreds of users creating thousands of pipelines, I'd separate the CRD lifecycle. But for infrastructure automation and VM builds? The simpler approach wins.

## Part Two: The Helm Chart Was a Lie

A few days later, I tried to actually configure Tekton with a `TektonConfig`. This is where things went sideways. Spectacularly.

### The Webhook Naming Bug

When I created the `TektonConfig`, Flux reported:

```
TektonConfig/config dry-run failed: admission webhook "webhook.operator.tekton.dev" denied the request
```

Dug into it - the webhook was looking for a service named `tekton-operator-webhook`, but the Helm chart created a service named `tekton-operator-tekton-operator-webhook`. Classic Helm double-naming bug where the release name (`tekton-operator`) gets concatenated with the chart's internal naming (`tekton-operator-webhook`). I considered a few options for dealing with this, and none of them seemed particularly appealing.

### Ditching Helm for Raw Manifests

The official Tekton installation docs don't even use Helm. They just do:

```bash
kubectl apply -f https://storage.googleapis.com/tekton-releases/operator/latest/release.yaml
```

Fine. Let's use the official manifests. The tektoncd/operator repo at v0.77.0 has a nice Kustomize structure, so I updated my Flux Kustomization to point to the tekton-operator GitRepository at that path.

### The `ko://` Nightmare

Pods started spinning up, but:

```
State:     Waiting
  Reason:  InvalidImageName
Events:
  Warning  InspectFailed  kubelet  Failed to apply default image tag
    "ko://github.com/tektoncd/operator/cmd/kubernetes/webhook":
    couldn't parse image name: invalid reference format
```

The image field was literally `ko://github.com/tektoncd/operator/cmd/kubernetes/webhook`.

What. The. Fuck.

Turns out the repo source manifests are meant to be processed by [ko](https://ko.build/), a tool that builds Go containers and replaces these placeholder URLs with actual container image references. The "release" artifacts on GCS have real images like `gcr.io/tekton-releases/...`, but the repo source files are just templates.

I grabbed the wrong thing. The repo isn't what you deploy. The repo is what you *build* to create what you deploy.

### Vendoring the Release Manifest

Fine. Downloaded the actual release manifest:

```bash
curl -sL "https://storage.googleapis.com/tekton-releases/operator/previous/v0.77.0/release.yaml" \
  -o infrastructure/tekton/operator/manifests/release.yaml
```

Updated the Flux structure to vendor the manifest:

```
infrastructure/tekton/operator/
├── flux-kustomization.yaml
├── kustomization.yaml
├── operator-install.yaml          # Points to manifests/
└── manifests/
    ├── kustomization.yaml
    └── release.yaml               # Vendored v0.77.0 release
```

Pushed. Reconciled. Operator finally deployed with real container images.

### Stuck CRDs During Reinstall

Of course it wasn't that simple. The old Helm installation left behind CRDs with finalizers. One CRD (`tektoninstallersets.operator.tekton.dev`) was stuck in `Terminating` state, blocking the new installation.

The culprit: an orphaned `TektonInstallerSet` resource with its own finalizer that was blocking the CRD from being deleted, which was blocking Flux from applying the new manifests.

```bash
# Nuclear option: remove the finalizer
kubectl patch tektoninstallerset validating-mutating-webhook-pknjj \
  -p '{"metadata":{"finalizers":[]}}' --type=merge
```

CRD finished deleting. New installation proceeded.

### TektonConfig Schema Fun

Now I needed to actually configure Tekton with a `TektonConfig`. Tried to be clever with settings like `disable-creds-init` and replica counts. Webhook rejected it:

```
unknown field "disable-creds-init"
```

`kubectl explain tektonconfig.spec` returned nothing useful because the CRD uses `x-kubernetes-preserve-unknown-fields: true`. Had to look at the actual example in the repo:

```yaml
apiVersion: operator.tekton.dev/v1alpha1
kind: TektonConfig
metadata:
  name: config
spec:
  profile: all
  targetNamespace: tekton-pipelines
```

That's it. The minimal config is *very* minimal. My elaborate config with custom options was using fields that don't exist in v0.77.0.

### Profile: all Means ALL

Used `profile: all`. Seemed reasonable. But "all" includes TektonResult, which needs a PostgreSQL database I don't have. Components kept failing because Result couldn't reconcile.

The fix was disabling the components I don't want:

```yaml
spec:
  profile: all
  targetNamespace: tekton-pipelines
  result:
    disabled: true  # Needs PostgreSQL
  chain:
    disabled: true  # Needs signing infrastructure
```

Finally. TektonConfig applied. Components deployed. Dashboard running.

### Exposing the Dashboard

The operator manages the Dashboard service, so I can't just change it to a LoadBalancer (it would get reverted). Created a separate LoadBalancer service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: tekton-dashboard-lb
  namespace: tekton-pipelines
  annotations:
    external-dns.alpha.kubernetes.io/hostname: tekton-dashboard.goldentooth.net
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: dashboard
    app.kubernetes.io/component: dashboard
  ports:
    - name: http
      port: 80
      targetPort: 9097
```

Dashboard is now accessible at `http://tekton-dashboard.goldentooth.net`.
