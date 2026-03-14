# Chaos Mesh: Replacing Litmus Because MongoDB Was Doing Nothing at 38% CPU

## The Audit

Three months ago, I deployed [Litmus](https://litmuschaos.io) into the cluster ([entry 085](./085_litmus.md)). That was a whole thing — hours of fighting ARM64 MongoDB incompatibilities, heterogeneous node placement, Bitnami image shenanigans. I was so happy when it finally worked that I... never actually ran an experiment.

Not one.

Ninety-one days. Zero experiments. Four pods sitting there. A full MongoDB replica set, faithfully consuming resources to store nothing.

I discovered this during a broader cluster cleanup session. A `KubeAPIErrorBudgetBurn` alert fired, which led me down a long path of investigating etcd I/O pressure on the control plane's SD cards. While auditing workload overhead, I found that Tekton had 39 leases generating constant PUT traffic with 0 pipelines, KubeVirt had 22 pods with 0 VMs, and Litmus was just... sitting there. Being expensive.

The bit that made me snap was checking `kubectl top pod` on the Litmus namespace:

```
litmus-auth-server-db74677b9-h5lk7   8m    95Mi
litmus-frontend-78bd5df698-6kb57     1m    42Mi
litmus-mongodb-0                     147m   180Mi
litmus-server-7ddb7b874f-64gtv       12m    131Mi
```

147 millicores. Just MongoDB. Just vibing. Doing absolutely nothing. Its CPU limit was 500m and it was throttling at 38%.

Jesus Christ.

## Why Litmus Was Wrong for This Cluster

In retrospect, Litmus ChaosCenter was always overkill here. It's designed for teams — a web UI for designing experiments, a MongoDB backend for storing execution history, authentication servers, GraphQL APIs. That's great if you have a team of SREs running coordinated chaos engineering campaigns against production.

I have 16 Raspberry Pis and a dream.

The ChaosCenter architecture meant I was running:
- **MongoDB** (via Bitnami, as a replica set even though there's one member) — the storage hog
- **GraphQL server** — orchestration API nobody was calling
- **Auth server** — authentication nobody was using
- **Frontend** — a React app nobody was viewing

All pinned to Pi 5 nodes because MongoDB requires ARMv8.2-A CPU instructions. All consuming resources. All doing nothing.

## Enter Chaos Mesh

[Chaos Mesh](https://chaos-mesh.org/) is a CNCF Incubating project that takes a fundamentally different approach: everything is a CRD.

No database. No UI server (optional, and I'm not deploying it). No auth layer. State lives where Kubernetes state already lives — in etcd, as Custom Resources. The controller-manager watches for chaos CRDs and orchestrates fault injection through the chaos-daemon DaemonSet.

That's it. That's the whole architecture:

1. **Controller Manager** — watches chaos CRDs, coordinates experiments
2. **Chaos Daemon** — runs on target nodes, performs the actual fault injection (network chaos via `tc`, I/O faults, process manipulation, etc.)
3. **CRDs** — 23 of them, covering everything from `PodChaos` to `NetworkChaos` to `IOChaos` to `KernelChaos`

This fits my GitOps workflow perfectly. Define an experiment as YAML, commit it to the repo, Flux applies it. Or `kubectl apply` it directly if I just want to break something right now. No UI needed.

## The Swap

### Ripping Out Litmus

First, surgical removal from the gitops repo:

```
# Deleted entirely
infrastructure/litmus/release.yaml
infrastructure/litmus/namespace.yaml
infrastructure/litmus/admin-secret.yaml
infrastructure/litmus/kustomization.yaml
infrastructure/gateway/routes/litmus.yaml

# Modified
infrastructure/kustomization.yaml          → removed litmus, added chaos-mesh
infrastructure/gateway/routes/kustomization.yaml  → removed litmus.yaml
infrastructure/gateway/reference-grant.yaml       → removed litmus namespace
```

Then cluster cleanup:

```
$ kubectl delete helmrelease litmus -n litmus
helmrelease.helm.toolkit.fluxcd.io "litmus" deleted

$ kubectl delete helmrepository litmuschaos -n flux-system
helmrepository.source.toolkit.fluxcd.io "litmuschaos" deleted

$ kubectl delete pvc datadir-litmus-mongodb-0 -n litmus
persistentvolumeclaim "datadir-litmus-mongodb-0" deleted

$ kubectl delete crd chaosexperiments.litmuschaos.io
customresourcedefinition.apiextensions.k8s.io "chaosexperiments.litmuschaos.io" deleted

$ kubectl delete ns litmus
namespace "litmus" deleted
```

Goodbye, old friend. We barely knew each other.

### Deploying Chaos Mesh

The new gitops structure:

```
infrastructure/chaos-mesh/
├── kustomization.yaml
├── namespace.yaml      # privileged PSA (chaos daemons need it)
├── repository.yaml     # charts.chaos-mesh.org
└── release.yaml        # The Helm release
```

The release values are minimal:

```yaml
values:
  controllerManager:
    replicaCount: 1
    nodeSelector:
      cpu.arch: armv8.2-a
    resources:
      requests:
        cpu: 25m
        memory: 128Mi
      limits:
        cpu: 250m
        memory: 512Mi

  chaosDaemon:
    nodeSelector:
      cpu.arch: armv8.2-a
    resources:
      requests:
        cpu: 25m
        memory: 64Mi
      limits:
        cpu: 250m
        memory: 256Mi

  dashboard:
    create: false

  dnsServer:
    create: false
```

One controller. Daemons only on Pi 5 nodes. No dashboard. No DNS server (that's only for `DNSChaos` experiments, enable it when I need it).

Push, wait for Flux, and:

```
$ kubectl get pods -n chaos-mesh -o wide
NAME                                        READY   STATUS    NODE
chaos-controller-manager-57fd584568-wkj4s   1/1     Running   norcross
chaos-daemon-67qws                          1/1     Running   manderly
chaos-daemon-hh4xd                          1/1     Running   norcross
chaos-daemon-xscmv                          1/1     Running   payne
chaos-daemon-xvmd5                          1/1     Running   oakheart
```

Five pods. All running. 23 CRDs installed.

## First Blood

Time to actually do the thing I never did with Litmus — run a chaos experiment.

Target: httpbin. It's expendable, it's simple, and it'll prove the system works.

First, confirm it's alive:

```
$ kubectl run curl-test --rm -it --restart=Never --image=busybox \
    -- wget -qO- http://httpbin.httpbin.svc.cluster.local/get
{
  "args": {},
  "headers": {
    "Host": ["httpbin.httpbin.svc.cluster.local"],
    "User-Agent": ["Wget"]
  },
  "method": "GET",
  "origin": "10.244.19.205",
  "url": "http://httpbin.httpbin.svc.cluster.local/get"
}
```

Now kill it:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: kill-httpbin
  namespace: chaos-mesh
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - httpbin
    labelSelectors:
      app: httpbin
```

```
$ kubectl apply -f - <<< '...'
podchaos.chaos-mesh.org/kill-httpbin created
```

Immediately:

```
$ kubectl get podchaos -n chaos-mesh kill-httpbin -o yaml | grep -A5 "phase:"
      phase: Injected

$ kubectl get events -n httpbin --sort-by='.lastTimestamp' | tail -5
Normal   Killing            pod/httpbin-76fcc48b8-9gvt5    Stopping container httpbin
Normal   SuccessfulCreate   replicaset/httpbin-76fcc48b8   Created pod: httpbin-76fcc48b8-bskwx
Normal   Pulling            pod/httpbin-76fcc48b8-bskwx    Pulling image "mccutchen/go-httpbin"
Normal   Pulled             pod/httpbin-76fcc48b8-bskwx    Successfully pulled image in 573ms
Normal   Started            pod/httpbin-76fcc48b8-bskwx    Started container httpbin
```

Pod killed. New pod up in under a second. Service recovered. The Deployment's replica set did exactly what it's supposed to do.

Verify:

```
$ kubectl run curl-test2 --rm -it --restart=Never --image=busybox \
    -- wget -qO- http://httpbin.httpbin.svc.cluster.local/status/200
```

200 OK. Clean.

```
$ kubectl delete podchaos kill-httpbin -n chaos-mesh
podchaos.chaos-mesh.org "kill-httpbin" deleted
```

Experiment complete. It was anticlimactic in the best possible way.

## The Numbers

Resource comparison:

| What | Pods | CPU (idle) | Memory |
|------|------|-----------|--------|
| **Litmus** | 4 (+ MongoDB StatefulSet) | ~168m | ~448Mi |
| **Chaos Mesh** | 5 (1 controller + 4 daemons) | ~9m | ~61Mi |

That's a 95% reduction in CPU and 86% reduction in memory. And Chaos Mesh actually _did something_ within 5 minutes of being deployed, which is more than Litmus managed in 91 days. To be fair, that's on me. But also, if using a tool requires fighting MongoDB's ARM64 issues just to get started, maybe the tool is the problem.

## What's Next

The chaos-daemons are currently restricted to Pi 5 nodes. For `PodChaos` experiments (pod-kill, pod-failure), that doesn't matter — the controller handles those via the Kubernetes API. But for the interesting stuff — network latency injection, I/O faults, kernel chaos — the daemon needs to be on the same node as the target. I'll expand the node selector when I start running those experiments.

The 23 CRDs are sitting there ready:

- `PodChaos` — kill, failure, container kill
- `NetworkChaos` — latency, loss, corruption, partition, bandwidth
- `IOChaos` — latency, fault, attribute override for filesystem operations
- `StressChaos` — CPU and memory stress
- `HTTPChaos` — HTTP request/response manipulation
- `TimeChaos` — clock skew
- `KernelChaos` — kernel fault injection

The plan is to build out a library of experiments in the gitops repo and run them as part of a regular chaos engineering practice. Scheduled experiments via `Schedule` CRDs, targeted at specific services during non-peak hours (not that there are peak hours on a homelab, but establishing good habits).

But first I have to make something worth breaking. The httpbin kill was satisfying in a primal way, but it's not exactly testing system resilience. That comes next.
