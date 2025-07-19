# Installing Argo CD

GitOps is a methodology based around treating IaC stored in Git as a source of truth for the desired state of the infrastructure. Put simply, whatever you push to `main` becomes the desired state and your IaC systems, whether they be Terraform, Ansible, etc, will be invoked to bring the actual state into alignment.

Argo CD is a popular system for implementing GitOps with Kubernetes. It can observe a Git repository for changes and react to those changes accordingly, creating/destroying/replacing resources as needed within the cluster.

Argo CD is a large, complicated application in its own right; its Helm chart is thousands of lines long. I'm not trying to learn it all right now, and fortunately, I have a fairly simple structure in mind.

I'll install Argo CD via a new Ansible [playbook](https://github.com/goldentooth/cluster/blob/main/playbooks/install_argocd.yaml) and [role](https://github.com/goldentooth/cluster/tree/main/roles/goldentooth.install_argocd) that use Helm, which we set up in the last section.

None of this is particularly complex, but I'll document some of my values overrides here:

```yaml
# I've seen a mix of `argocd` and `argo-cd` scattered around. I preferred
# `argocd`, but I will shift to `argo-cd` where possible to improve
# consistency.
#
# EDIT: The `argocd` CLI tool appears to be broken and does not allow me to
# override the names of certain components when port forwarding.
# See https://github.com/argoproj/argo-cd/issues/16266 for details.
# As a result, I've gone through and reverted my changes to standardize as much
# as possible on `argocd`. FML.
nameOverride: 'argocd'
global:
  # This evaluates to `argocd.goldentooth.hellholt.net`.
  domain: "{{ argocd_domain }}"
  # Add Prometheus scrape annotations to all metrics services. This can
  # be used as an alternative to the ServiceMonitors.
  addPrometheusAnnotations: true
  # Default network policy rules used by all components.
  networkPolicy:
    # Create NetworkPolicy objects for all components; this is currently false
    # but I think I'd like to create these later.
    create: false
    # Default deny all ingress traffic; I want to improve security, so I hope
    # to enable this later.
    defaultDenyIngress: false
configs:
  secret:
    createSecret: true
    # Specify a password. I store an "easy" password, which is in my muscle
    # memory, so I'll use that for right now.
    argocdServerAdminPassword: "{{ vault.easy_password | password_hash('bcrypt') }}"
  # Refer to the repositories that host our applications.
  repositories:
    # This is the main (and likely only) one.
    gitops:
      type: 'git'
      name: 'gitops'
      # This turns out to be https://github.com/goldentooth/gitops.git
      url: "{{ argocd_app_repo_url }}"

redis-ha:
  # Enable Redis high availability.
  enabled: true

controller:
  # The HA configuration keeps this at one, and I don't see a reason to change.
  replicas: 1

server:
  # Enable
  autoscaling:
    enabled: true
    # This immediately scaled up to 3 replicas.
    minReplicas: 2
  # I'll make this more secure _soon_.
  extraArgs:
    - '--insecure'
  # I don't have load balancing set up yet.
  service:
    type: 'ClusterIP'

repoServer:
  autoscaling:
    enabled: true
    minReplicas: 2

applicationSet:
  replicas: 2
```

![Pods in the Argo CD namespace](./images/017_argocd_pods.png)

## Installation Architecture

The Argo CD installation uses a sophisticated Helm-based approach with the following components:

- **Chart Version**: 7.1.5 from the official Argo repository (`https://argoproj.github.io/argo-helm`)
- **CLI Installation**: ARM64-specific Argo CD CLI installed to `/usr/local/bin/argocd`
- **Namespace**: Dedicated `argocd` namespace with proper resource isolation
- **Deployment Scope**: Runs once on control plane nodes for efficient resource usage

### High Availability Configuration

The installation implements enterprise-grade high availability:

**Redis High Availability**:
```yaml
redis-ha:
  enabled: true
```

**Component Scaling**:
- **Server**: Autoscaling enabled with minimum 2 replicas for redundancy
- **Repo Server**: Autoscaling enabled with minimum 2 replicas for Git repository operations
- **Application Set Controller**: 2 replicas for ApplicationSet management
- **Controller**: 1 replica (following HA recommendations for the core controller)

This configuration ensures that Argo CD remains operational even during node failures or maintenance.

### Security and Authentication

**Admin Authentication**:
The cluster uses bcrypt-hashed passwords stored in the encrypted Ansible vault:

```yaml
argocdServerAdminPassword: "{{ secret_vault.easy_password | password_hash('bcrypt') }}"
```

**GitHub Integration**:
For private repository access, the installation creates a Kubernetes secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-token
  namespace: argocd
data:
  token: "{{ secret_vault.github_token | b64encode }}"
```

**Current Security Posture**:
- Server configured with `--insecure` flag (temporary for initial setup)
- Network policies prepared but not yet enforced
- RBAC relies on default admin access patterns

### Service and Network Integration

**LoadBalancer Configuration**:
Unlike the basic ClusterIP shown in the values, the actual deployment uses:

```yaml
service:
  type: LoadBalancer
  annotations:
    external-dns.alpha.kubernetes.io/hostname: "argocd.{{ cluster.domain }}"
    external-dns.alpha.kubernetes.io/ttl: "60"
```

This integration provides:
- **MetalLB Integration**: Automatic IP address assignment from the `10.4.11.0/24` pool
- **External DNS**: Automatic DNS record creation for `argocd.goldentooth.net`
- **Public Access**: Direct access from the broader network infrastructure

### GitOps Implementation: App of Apps Pattern

The cluster implements the sophisticated "Application of Applications" pattern for managing GitOps workflows:

**AppProject Configuration**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: gitops-repo
spec:
  sourceRepos:
    - '*'  # Lab environment - all repositories allowed
  destinations:
    - namespace: '*'
       server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
```

**ApplicationSet Generator**:
The cluster uses GitHub SCM Provider generator to automatically discover and deploy applications:

```yaml
generators:
- scmProvider:
    github:
      organization: goldentooth
      labelSelector:
        matchLabels:
          gitops-repo: "true"
```

This pattern automatically creates Argo CD Applications for any repository in the goldentooth organization with the `gitops-repo` label.

### Application Standards and Sync Policies

**Standardized Sync Configuration**:
All applications follow consistent sync policies:

```yaml
syncPolicy:
  automated:
    prune: true      # Remove resources not in Git
    selfHeal: true   # Automatically fix configuration drift
  syncOptions:
    - Validate=true
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    - RespectIgnoreDifferences=true
    - ApplyOutOfSyncOnly=true
```

**Wave-based Deployment**:
Applications use `argocd.argoproj.io/wave` annotations for ordered deployment, ensuring dependencies are deployed before dependent services.

### Monitoring Integration

**Prometheus Integration**:
```yaml
global:
  addPrometheusAnnotations: true
```

This configuration ensures all Argo CD components expose metrics for the cluster's Prometheus monitoring stack, providing visibility into GitOps operations and performance.

### Current Application Portfolio

The GitOps system currently manages:
- **MetalLB**: Load balancer implementation
- **External Secrets**: Integration with HashiCorp Vault
- **Prometheus Node Exporter**: Node-level monitoring
- **Additional applications**: Automatically discovered via the ApplicationSet pattern

### Command Line Integration

The installation provides seamless CLI integration:

```bash
# Install Argo CD
goldentooth install_argo_cd

# Install managed applications
goldentooth install_argo_cd_apps
```

### Access Methods

**Web Interface Access**:
- **Production**: Direct access via `https://argocd.goldentooth.net` (LoadBalancer + External DNS)
- **Development**: Port forwarding via `kubectl -n argocd port-forward service/argocd-server 8081:443 --address 0.0.0.0`

After running the port-forward command on one of my control plane nodes, I'm able to view the web interface and log in. With the App of Apps pattern configured, the interface shows automatically discovered applications and their sync status.

The GitOps foundation is now established, enabling declarative application management across the entire cluster infrastructure.
