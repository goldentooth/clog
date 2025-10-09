# MetalLB (again)

I won't document this separately for obvious reasons, but I followed the excellent ["Getting Started"](https://fluxcd.io/flux/get-started/) and ["Ways of Structuring Your Repositories"](https://fluxcd.io/flux/guides/repository-structure/) documentation to get FluxCD set up with the [`gitops` repository](https://github.com/goldentooth/gitops/). Now I'm back to having some semblance of GitOps, although without any applications of note (though [`podinfo`](https://github.com/stefanprodan/podinfo) is really cool!).

So that brings us back to setting up MetalLB so that I can easily access Kubernetes services.

MetalLB was straightforward. Its namespace needs elevated privileges, but the Helm chart and repository definition were very straightforward:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: metallb-system
  labels:
    # Allow MetalLB speaker pods to use privileged capabilities
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: metallb
  namespace: flux-system
spec:
  interval: 30m
  targetNamespace: metallb-system
  chart:
    spec:
      chart: metallb
      version: "0.14.5"
      sourceRef:
        kind: HelmRepository
        name: metallb
        namespace: flux-system
  install:
    createNamespace: false
    crds: Create
    remediation:
      retries: 3
  upgrade:
    crds: CreateReplace
    remediation:
      retries: 3
  values:
    # Enable Prometheus metrics (ServiceMonitor disabled - requires Prometheus Operator)
    prometheus:
      serviceAccount: metallb-controller
      namespace: metallb-system
      serviceMonitor:
        enabled: false
    # Speaker configuration for L2 mode
    speaker:
      enabled: true
      tolerateMaster: true
      # Disable FRR for simple L2 mode
      frr:
        enabled: false
    # Controller configuration
    controller:
      enabled: true
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: metallb
  namespace: flux-system
spec:
  interval: 24h
  url: https://metallb.github.io/metallb
```

I placed that information in `infrastructure/metallb`, and then the following MetalLB configuration resources in `apps/metallb.yaml` to deploy subsequently:

```yaml
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: primary
  namespace: metallb-system
spec:
  addresses:
    - "10.4.11.0-10.4.15.254"
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: primary
  namespace: metallb-system
spec:
  ipAddressPools:
    - primary
```

My podinfo configuration looked like this:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: podinfo
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: master
  url: https://github.com/stefanprodan/podinfo
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 30m0s
  path: ./kustomize
  prune: true
  retryInterval: 2m0s
  sourceRef:
    kind: GitRepository
    name: podinfo
  targetNamespace: podinfo
  timeout: 3m0s
  wait: true
  patches:
    - patch: |-
        apiVersion: autoscaling/v2
        kind: HorizontalPodAutoscaler
        metadata:
          name: podinfo
        spec:
          minReplicas: 3
      target:
        name: podinfo
        kind: HorizontalPodAutoscaler
    - patch: |-
        - op: add
          path: /metadata/annotations/metallb.io~1address-pool
          value: default
        - op: replace
          path: /spec/type
          value: LoadBalancer
        - op: add
          path: /spec/externalTrafficPolicy
          value: Local
        - op: replace
          path: /spec/ports
          value:
            - port: 80
              targetPort: 9898
              protocol: TCP
              name: http
            - port: 9999
              targetPort: 9999
              protocol: TCP
              name: grpc
      target:
        kind: Service
        name: podinfo
```
