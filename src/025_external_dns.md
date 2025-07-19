# ExternalDNS

The workflow for accessing our `LoadBalancer` services ain't great.

If we deploy a new application, we need to run `kubectl -n <namespace> get svc` and read through a list to determine the IP address on which it's exposed. And that's not going to be stable; there's nothing at all guaranteeing that Argo CD will always be available at `http://10.4.11.1`.

Enter [ExternalDNS](https://github.com/kubernetes-sigs/external-dns). The idea is that we annotate our services with `external-dns.alpha.kubernetes.io/hostname: "argocd.my-cluster.my-domain.com"` and a DNS record will be created pointing to the actual IP address of the `LoadBalancer` service.

This is comparatively straightforward to configure if you host your DNS in one of the supported services. I host mine via AWS Route53, which is supported.

The complication is that we don't yet have a great way of managing secrets, so there's a manual step here that I find unpleasant, but we'll cross that bridge when we get to it.

## Architecture Overview

ExternalDNS creates a bridge between Kubernetes services and external DNS providers, enabling automatic DNS record management:

### DNS Infrastructure
- **Primary Domain**: `goldentooth.net` managed in AWS Route53
- **Zone ID**: `Z0736727S7ZH91VKK44A` (defined in Terraform)
- **Cluster Subdomain**: Services automatically get `<service>.goldentooth.net`
- **TTL Configuration**: Default 60 seconds for rapid updates during development

### Integration Points
- **MetalLB**: Provides LoadBalancer IPs from pool `10.4.11.0/24`
- **Route53**: AWS DNS service for public domain management
- **Argo CD**: GitOps deployment and lifecycle management
- **Terraform**: Infrastructure-as-code for Route53 zone and ACM certificates

## Helm Chart Configuration

Because of work we've done previously with Argo CD, we can just create [a new repository](https://github.com/goldentooth/external-dns) to deploy ExternalDNS within our cluster.

The ExternalDNS deployment is managed through a custom Helm chart with comprehensive configuration:

### Chart Structure (`Chart.yaml`)
```yaml
apiVersion: v2
name: external-dns
description: ExternalDNS for automatic DNS record management
type: application
version: 0.0.1
appVersion: "v0.14.2"
```

### Values Configuration (`values.yaml`)
```yaml
metadata:
  namespace: external-dns
  name: external-dns
  projectName: gitops-repo

spec:
  domainFilter: goldentooth.net
  version: v0.14.2
```

This configuration provides:
- **Namespace isolation**: Dedicated `external-dns` namespace
- **GitOps integration**: Part of the `gitops-repo` project for Argo CD
- **Domain scoping**: Only manages records for `goldentooth.net`
- **Version pinning**: Uses ExternalDNS v0.14.2 for stability

## Deployment Architecture

### Core Deployment Configuration

This has the following manifests:

**Deployment**: The deployment has several interesting features:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: external-dns
spec:
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
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
        - --aws-region=us-east-1
        env:
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

**Key Configuration Parameters**:
- **Provider**: `aws` for Route53 integration
- **Sources**: `service` (monitors Kubernetes LoadBalancer services)
- **Domain Filter**: `goldentooth.net` (restricts DNS management scope)
- **AWS Zone Type**: `public` (only manages public DNS records)
- **Registry**: `txt` (uses TXT records for ownership tracking)
- **Owner ID**: `external-dns-external-dns` (namespace-app format)
- **Region**: `us-east-1` (AWS region for Route53 operations)

### AWS Credentials Management

**Secret Configuration**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: external-dns
  namespace: external-dns
type: Opaque
data:
  credentials: |
    [default]
    aws_access_key_id = {{ secret_vault.aws.access_key_id | b64encode }}
    aws_secret_access_key = {{ secret_vault.aws.secret_access_key | b64encode }}
```

This setup:
- **Secure storage**: AWS credentials stored in Ansible vault
- **Minimal permissions**: IAM user with only Route53 zone modification rights
- **File-based auth**: Uses AWS credentials file format for compatibility
- **Namespace isolation**: Secret accessible only within `external-dns` namespace

### RBAC Configuration

**ServiceAccount**: Just adds a service account for ExternalDNS.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  namespace: external-dns
```

**ClusterRole**: Describes an ability to observe changes in services.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services", "endpoints", "pods", "nodes"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["extensions", "networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "watch", "list"]
```

**ClusterRoleBinding**: Binds the above cluster role and ExternalDNS.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: external-dns
```

**Permission Scope**:
- **Read-only access**: ExternalDNS cannot modify Kubernetes resources
- **Cluster-wide monitoring**: Can watch services across all namespaces
- **Resource types**: Services, endpoints, pods, nodes, and ingresses
- **Security principle**: Least privilege for DNS management operations

## Service Annotation Patterns

### Basic DNS Record Creation

Services use annotations to trigger DNS record creation:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  namespace: httpbin
  annotations:
    external-dns.alpha.kubernetes.io/hostname: httpbin.goldentooth.net
    external-dns.alpha.kubernetes.io/ttl: "60"
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: httpbin
```

**Annotation Functions**:
- **Hostname**: `external-dns.alpha.kubernetes.io/hostname` specifies the FQDN
- **TTL**: `external-dns.alpha.kubernetes.io/ttl` sets DNS record time-to-live
- **Automatic A record**: Points to MetalLB-allocated LoadBalancer IP
- **Automatic TXT record**: Ownership tracking with txt-owner-id

### Advanced Annotation Options

```yaml
annotations:
  # Multiple hostnames for the same service
  external-dns.alpha.kubernetes.io/hostname: "app.goldentooth.net,api.goldentooth.net"
  
  # Custom TTL for caching strategy
  external-dns.alpha.kubernetes.io/ttl: "300"
  
  # AWS-specific: Route53 weighted routing
  external-dns.alpha.kubernetes.io/aws-weight: "100"
  
  # AWS-specific: Health check configuration
  external-dns.alpha.kubernetes.io/aws-health-check-id: "12345678-1234-1234-1234-123456789012"
```

## DNS Record Lifecycle Management

### Record Creation Process

1. **Service Creation**: LoadBalancer service deployed with ExternalDNS annotations
2. **IP Allocation**: MetalLB assigns IP from configured pool (`10.4.11.0/24`)
3. **Service Discovery**: ExternalDNS watches Kubernetes API for service changes
4. **DNS Creation**: Creates A record pointing to LoadBalancer IP
5. **Ownership Tracking**: Creates TXT record for ownership verification

### Record Cleanup Process

1. **Service Deletion**: LoadBalancer service removed from cluster
2. **Change Detection**: ExternalDNS detects service removal event
3. **Ownership Verification**: Checks TXT record ownership before deletion
4. **DNS Cleanup**: Removes both A and TXT records from Route53
5. **IP Release**: MetalLB returns IP to available pool

### TXT Record Ownership System

ExternalDNS uses TXT records for safe multi-cluster DNS management:

```bash
# Example TXT record for ownership tracking
dig TXT httpbin.goldentooth.net

# Response includes:
# httpbin.goldentooth.net. 60 IN TXT "heritage=external-dns,external-dns/owner=external-dns-external-dns"
```

This prevents:
- **Record conflicts**: Multiple ExternalDNS instances managing same domain
- **Accidental deletion**: Only owner can modify/delete records
- **Split-brain scenarios**: Clear ownership prevents conflicting updates

## Integration with GitOps

### Argo CD Application Configuration

ExternalDNS is deployed via GitOps using the ApplicationSet pattern:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: gitops-repo
  namespace: argocd
spec:
  generators:
  - scmProvider:
      github:
        organization: goldentooth
        allBranches: false
        labelSelector:
          matchLabels:
            gitops-repo: "true"
  template:
    metadata:
      name: '{{repository}}'
    spec:
      project: gitops-repo
      source:
        repoURL: '{{url}}'
        targetRevision: '{{branch}}'
        path: .
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{repository}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

This provides:
- **Automatic deployment**: Changes to external-dns repository trigger redeployment
- **Namespace creation**: Automatically creates `external-dns` namespace
- **Self-healing**: Argo CD corrects configuration drift
- **Pruning**: Removes resources deleted from Git repository

### Repository Structure

```
external-dns/
├── Chart.yaml          # Helm chart metadata
├── values.yaml         # Configuration values
└── templates/
    ├── Deployment.yaml  # ExternalDNS deployment
    ├── ServiceAccount.yaml
    ├── ClusterRole.yaml
    ├── ClusterRoleBinding.yaml
    └── Secret.yaml      # AWS credentials (Ansible-templated)
```

## Monitoring and Troubleshooting

### Health Verification

```bash
# Check ExternalDNS pod status
kubectl -n external-dns get pods

# Monitor ExternalDNS logs
kubectl -n external-dns logs -l app=external-dns --tail=50

# Verify AWS credentials
kubectl -n external-dns exec deployment/external-dns -- cat /.aws/credentials

# Check service discovery
kubectl -n external-dns logs deployment/external-dns | grep "Creating record"
```

### DNS Record Validation

```bash
# Verify A record creation
dig A httpbin.goldentooth.net

# Check TXT record ownership
dig TXT httpbin.goldentooth.net

# Validate Route53 changes
aws route53 list-resource-record-sets --hosted-zone-id Z0736727S7ZH91VKK44A | jq '.ResourceRecordSets[] | select(.Name | contains("httpbin"))'
```

### Common Issues and Solutions

**Issue**: DNS records not created
- **Check**: Service has `type: LoadBalancer` and LoadBalancer IP is assigned
- **Verify**: ExternalDNS has RBAC permissions to read services
- **Debug**: Check ExternalDNS logs for AWS API errors

**Issue**: DNS records not cleaned up
- **Check**: TXT record ownership matches ExternalDNS txt-owner-id
- **Verify**: AWS credentials have Route53 delete permissions
- **Debug**: Monitor ExternalDNS logs during service deletion

**Issue**: Multiple DNS entries for same service
- **Check**: Only one ExternalDNS instance should manage each domain
- **Verify**: txt-owner-id is unique across clusters
- **Fix**: Use different owner IDs for different environments

## Integration Examples

### Argo CD Access

A few minutes after pushing changes to the repository, we can reach Argo CD via https://argocd.goldentooth.net/.

**Service Configuration**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    external-dns.alpha.kubernetes.io/hostname: argocd.goldentooth.net
    external-dns.alpha.kubernetes.io/ttl: "60"
spec:
  type: LoadBalancer
  ports:
  - port: 443
    targetPort: 8080
    protocol: TCP
    name: https
  selector:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: argocd-server
```

This automatically creates:
- **A record**: `argocd.goldentooth.net → 10.4.11.1` (MetalLB-assigned IP)
- **TXT record**: Ownership tracking for safe management
- **60-second TTL**: Rapid DNS propagation for development workflows

The combination of MetalLB for LoadBalancer IP allocation and ExternalDNS for automatic DNS management creates a seamless experience where services become accessible via friendly DNS names without manual intervention, enabling true infrastructure-as-code patterns for both networking and DNS.
