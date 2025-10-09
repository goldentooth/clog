# External DNS (again)

Now that the IP-layer stuff is set up, I need DNS-layer stuff. I want to be able to request `https://<service>.goldentooth.net/` and access that service.

Fortunately, I already had this in place once before, so I knew how to do it. Unlike MetalLB, which uses a Helm chart, I opted for a plain Kubernetes deployment for External-DNS. This gives me finer control over the configuration and keeps things simpler for this use case.

## Infrastructure Setup

The External-DNS deployment lives in `infrastructure/external-dns` and consists of several standard Kubernetes resources:

### Namespace and Service Account

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: external-dns
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  namespace: external-dns
  labels:
    app.kubernetes.io/name: external-dns
```

### RBAC Configuration

External-DNS needs cluster-wide read access to watch for services, ingresses, and other resources that might need DNS records:

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
  labels:
    app.kubernetes.io/name: external-dns
rules:
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get","watch","list"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get","watch","list"]
  - apiGroups: ["networking","networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get","watch","list"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get","watch","list"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get","watch","list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
  labels:
    app.kubernetes.io/name: external-dns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
  - kind: ServiceAccount
    name: external-dns
    namespace: external-dns
```

### Deployment

The deployment itself runs a single instance of External-DNS (using `Recreate` strategy to avoid conflicts) and configures it to watch services and update AWS Route 53:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: external-dns
  labels:
    app.kubernetes.io/name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app.kubernetes.io/name: external-dns
  template:
    metadata:
      labels:
        app.kubernetes.io/name: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
        - name: external-dns
          image: registry.k8s.io/external-dns/external-dns:v0.14.2
          args:
            - --source=service
            - --domain-filter=goldentooth.net
            - --provider=aws
            - --aws-zone-type=public
            - --registry=txt
            - --txt-owner-id=external-dns-external-dns
            - --log-level=debug
          env:
            - name: AWS_DEFAULT_REGION
              value: us-east-1
            - name: AWS_SHARED_CREDENTIALS_FILE
              value: /.aws/credentials
          volumeMounts:
            - name: aws-credentials
              mountPath: /.aws
              readOnly: true
      volumes:
        - name: aws-credentials
          secret:
            secretName: external-dns
```

### AWS Credentials

The deployment mounts AWS credentials from a SOPS-encrypted secret (stored in `secret.yaml`). This secret contains an AWS credentials file with permissions to update Route 53 records for the goldentooth.net zone.

I'm new to Sops, but really digging it so far. It's far nicer than depending (hackily) on the Ansible vault I was using before.

With this all in place, services can be annotated to create DNS records for them. I updated the `podinfo` Kustomization patches to add that hostname:

```yaml
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
        - op: add
          path: /metadata/annotations/external-dns.alpha.kubernetes.io~1hostname
          value: podinfo.goldentooth.net
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

and it works:

```bash
$ curl http://podinfo.goldentooth.net/
{
  "hostname": "podinfo-6fd9b57958-7sr4v",
  "version": "6.9.2",
  "revision": "e86405a8674ecab990d0a389824c7ebbd82973b5",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v6.9.2",
  "goos": "linux",
  "goarch": "arm64",
  "runtime": "go1.25.1",
  "num_goroutine": "8",
  "num_cpu": "4"
}
```
