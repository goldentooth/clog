# KubeVirt: Virtual Machines in Kubernetes

I've been wanting to run VMs on the cluster for a while now. Not because I have any immediate need for them, but because KubeVirt is one of those technologies that's just... cool? The idea of managing virtual machines as Kubernetes resources, using the same GitOps workflows, the same kubectl commands – it's elegant in a way that appeals to the part of my brain that got me into infrastructure in the first place.

Plus, having the ability to spin up VMs on demand could be useful for testing OS-level stuff, running workloads that don't containerize well, or just experimenting with things that need a full VM environment.

## The Problem: Stuck Kustomizations

I added KubeVirt to my Flux setup and pushed the changes. After a few minutes, I checked on things:

```bash
$ flux get kustomizations -w
NAME                      REVISION              SUSPENDED    READY    MESSAGE
apps                      main@sha1:e98a5de0    False        False    dependency 'flux-system/infrastructure' is not ready
flux-system               main@sha1:e98a5de0    False        True     Applied revision: main@sha1:e98a5de0
httpbin                   main@sha1:e98a5de0    False        True     Applied revision: main@sha1:e98a5de0
infrastructure            main@sha1:e98a5de0    False        Unknown  Reconciliation in progress
kubevirt-cdi              main@sha1:e98a5de0    False        Unknown  Reconciliation in progress
kubevirt-instance         main@sha1:e98a5de0    False        False    dependency 'flux-system/kubevirt-cdi' is not ready
kubevirt-operator         main@sha1:e98a5de0    False        True     Applied revision: main@sha1:e98a5de0
```

Not great. `kubevirt-cdi` was stuck in "Reconciliation in progress" with an "Unknown" ready status. This caused a cascade of failures – `kubevirt-instance` couldn't start because it depends on `kubevirt-cdi`, and `apps` couldn't start because it depends on `infrastructure`.

## The Debugging Journey

Time to dig in. First, I checked Flux events to see what was actually happening:

```bash
$ flux events
...
3m21s    Warning    HealthCheckFailed    Kustomization/kubevirt-cdi    health check failed after 9m30s: timeout waiting for: [Deployment/cdi/cdi-operator status: 'InProgress']
```

Ah! So the `cdi-operator` Deployment in the `cdi` namespace was stuck. The health check was timing out because the deployment never became healthy.

Let's look at the deployment itself:

```bash
$ kubectl -n cdi get deployments
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
cdi-operator   0/1     1            0           28m
```

0/1 ready. Not good. What about the pods?

```bash
$ kubectl -n cdi get all
NAME                                READY   STATUS             RESTARTS         AGE
pod/cdi-operator-797944b474-ql7fz   0/1     CrashLoopBackOff   10 (2m22s ago)   29m
```

CrashLoopBackOff! Now we're getting somewhere. Let me describe the pod to see what's going on:

```bash
$ kubectl -n cdi describe pod cdi-operator-797944b474-ql7fz
...
State:           Waiting
  Reason:        CrashLoopBackOff
Last State:      Terminated
  Reason:        Error
  Exit Code:     255
...
Events:
  Warning  BackOff    4m4s (x115 over 29m)  kubelet    Back-off restarting failed container
```

Exit code 255, constantly restarting. The pod events show it's been failing for 29 minutes. Let's check the logs:

```bash
$ kubectl -n cdi logs cdi-operator-797944b474-ql7fz
exec /usr/bin/cdi-operator: exec format error
```

**There it is.** "exec format error" – that's the kernel telling me it can't execute the binary because it was compiled for a different CPU architecture.

The pod was scheduled on `jast`, one of my Raspberry Pi 4B nodes running ARM64. The container image must be built for AMD64/x86_64, not ARM64.

## Understanding the Architecture Mismatch

Of course, container images are architecture-specific unless they're built as multi-arch images with manifest lists. When you pull an image, the container runtime tries to find a manifest for your architecture. If it doesn't exist, you might get the wrong architecture anyway, and then... exec format error.

KubeVirt CDI does publish ARM64 images, but they're tagged differently. Instead of using manifest lists where the same tag works for all architectures, they use separate tags like `v1.63.1-arm64`. Not a big deal.

## The Solution: Two Problems, Two Fixes

I had attempted to override the images:

```yaml
images:
  - name: quay.io/kubevirt/cdi-operator
    newTag: v1.63.1-arm64
  - name: quay.io/kubevirt/cdi-controller
    newTag: v1.63.1-arm64
  # ... etc
```

But that didn't work. Turns out I missed something - my upstream kustomization at `gitops/infrastructure/kubevirt/upstream-cdi/kustomization.yaml` was still pulling the v1.61.1 manifest:

```yaml
resources:
  - https://github.com/kubevirt/containerized-data-importer/releases/download/v1.61.1/cdi-operator.yaml
```

Even with image overrides, that manifest had environment variables hardcoded to v1.61.1 images, and v1.61.1-arm64 tags don't exist in the registry.

Committed, pushed, and forced a reconciliation:

```bash
flux reconcile kustomization kubevirt-cdi --with-source
```

## Does It Work?

A minute later:

```bash
$ flux get kustomizations
NAME                      REVISION                 SUSPENDED    READY    MESSAGE
apps                      main@sha1:773fbac5       False        True     Applied revision: main@sha1:773fbac5
flux-system               main@sha1:773fbac5       False        True     Applied revision: main@sha1:773fbac5
httpbin                   main@sha1:773fbac5       False        True     Applied revision: main@sha1:773fbac5
infrastructure            main@sha1:773fbac5       False        True     Applied revision: main@sha1:773fbac5
kubevirt-cdi              main@sha1:773fbac5       False        True     Applied revision: main@sha1:773fbac5
kubevirt-instance         main@sha1:773fbac5       False        True     Applied revision: main@sha1:773fbac5
kubevirt-operator         main@sha1:773fbac5       False        True     Applied revision: main@sha1:773fbac5
```

All green! Everything reconciled successfully. The cascade of dependencies resolved – `kubevirt-cdi` became ready, which unblocked `kubevirt-instance`, which unblocked `infrastructure`, which unblocked `apps`.
