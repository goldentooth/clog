# The "Incubator" GitOps Application

Previously, we discussed GitOps and how Argo CD provides a platform for implementing GitOps for Kubernetes.

As mentioned, the general idea is to have some Git repository somewhere that defines an application. We create a corresponding resource in Argo CD to represent that application, and Argo CD will henceforth watch the repository and make changes to the running application as needed.

What does the repository actually include? Well, it might be a Helm chart, or a kustomization, or raw manifests, etc. Pretty much anything that could be done in Kubernetes.

Of course, setting this up involves some manual work; you need to actually create the application within Argo CD and, if you want it to hang around, you need to presumably commit that resource to some version control system somewhere. We of course want to be careful who has access to that repository, though, and we might not want engineers to have access to Argo CD itself. So suddenly there's a rather uncomfortable amount of work and coupling in all of this.

## GitOps Deployment Patterns

### Traditional Application Management Challenges

**Manual application creation**:
- Platform engineers must create Argo CD Application resources manually
- Direct access to Argo CD UI required for application management
- Configuration drift between different environments
- Difficulty managing permissions and access control at scale

**Repository proliferation**:
- Each application requires its own repository or subdirectory
- Inconsistent structure and standards across teams
- Complex permission management across multiple repositories
- Operational overhead for maintaining repository access

### The App-of-Apps Pattern

A common pattern in Argo CD is the "app-of-apps" pattern. This is simply an Argo CD application pointing to a repository that contains other Argo CD applications. Thus you can have a single application created for you by the principal platform engineer, and you can turn it into fifty or a hundred finely grained pieces of infrastructure that said principal engineer doesn't have to know about 🙂

(If they haven't configured the security settings carefully, it can all just be your little secret 😉)

**App-of-Apps Architecture**:
```
Root Application (managed by platform team)
├── Application 1 (e.g., monitoring stack)
├── Application 2 (e.g., ingress controllers)
├── Application 3 (e.g., security tools)
└── Application N (e.g., developer applications)
```

**Benefits of App-of-Apps**:
- **Single entry point**: Platform team manages one root application
- **Delegated management**: Development teams control their applications
- **Hierarchical organization**: Logical grouping of related applications
- **Simplified bootstrapping**: New environments start with root application

**Limitations of App-of-Apps**:
- **Resource proliferation**: Each application creates an Application resource
- **Dependency management**: Complex inter-application dependencies
- **Scaling challenges**: Manual management of hundreds of applications
- **Limited templating**: Difficult to apply consistent patterns

### ApplicationSet Pattern (Modern Approach)

A (relatively) new construct in Argo CD is the ApplicationSet construct, which seeks to more clearly define how applications are created and fix the problems with the "app-of-apps" approach. That's the approach we will take in this cluster for mature applications.

**ApplicationSet Architecture**:
```yaml
ApplicationSet (template-driven)
├── Generator (Git directories, clusters, pull requests)
├── Template (Application template with parameters)
└── Applications (dynamically created from template)
```

**ApplicationSet Generators**:
- **Git Generator**: Scans Git repositories for directories or files
- **Cluster Generator**: Creates applications across multiple clusters
- **List Generator**: Creates applications from predefined lists
- **Matrix Generator**: Combines multiple generators for complex scenarios

**Example ApplicationSet Configuration**:
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

## The Incubator Project Strategy

Given that we're operating in a lab environment, we can use the "app-of-apps" approach for the **Incubator**, which is where we can try out new configurations. We can give it fairly unrestricted access while we work on getting things to deploy correctly, and then lock things down as we zero in on a stable configuration.

### Development vs Production Patterns

**Incubator (Development)**:
- **App-of-Apps pattern**: Manual application management for experimentation
- **Permissive security**: Broad access for rapid prototyping
- **Flexible structure**: Accommodate diverse application types
- **Quick iteration**: Fast deployment and testing cycles

**Production (Mature Applications)**:
- **ApplicationSet pattern**: Template-driven automation at scale
- **Restrictive security**: Principle of least privilege
- **Standardized structure**: Consistent patterns and practices
- **Controlled deployment**: Change management and approval processes

But meanwhile, we'll create an `AppProject` manifest for the Incubator:

```yaml
---
apiVersion: 'argoproj.io/v1alpha1'
kind: 'AppProject'
metadata:
  name: 'incubator'
  # Argo CD resources need to deploy into the Argo CD namespace.
  namespace: 'argocd'
  finalizers:
    - 'resources-finalizer.argocd.argoproj.io'
spec:
  description: 'Goldentooth incubator project'
  # Allow manifests to deploy from any Git repository.
  # This is an acceptable security risk because this is a lab environment
  # and I am the only user.
  sourceRepos:
    - '*'
  destinations:
    # Prevent any resources from deploying into the kube-system namespace.
    - namespace: '!kube-system'
      server: '*'
    # Allow resources to deploy into any other namespace.
    - namespace: '*'
      server: '*'
  clusterResourceWhitelist:
    # Allow any cluster resources to deploy.
    - group: '*'
      kind: '*'
```

As mentioned before, this is very permissive. It only slightly differs from the default project by preventing resources from deploying into the `kube-system` namespace.

We'll also create an `Application` manifest:

```yaml
apiVersion: 'argoproj.io/v1alpha1'
kind: 'Application'
metadata:
  name: 'incubator'
  namespace: 'argocd'
  labels:
    name: 'incubator'
    managed-by: 'argocd'
spec:
  project: 'incubator'
  source:
    repoURL: "https://github.com/goldentooth/incubator.git"
    path: './'
    targetRevision: 'HEAD'
  destination:
    server: 'https://kubernetes.default.svc'
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - Validate=true
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
      - RespectIgnoreDifferences=true
      - ApplyOutOfSyncOnly=true
```

That's sufficient to get it to pop up in the Applications view in Argo CD.

![Argo CD Incubator](./images/018_argocd_incubator.png)

## AppProject Configuration Deep Dive

### Security Boundary Configuration

The AppProject resource provides security boundaries and policy enforcement:

```yaml
spec:
  description: 'Goldentooth incubator project'
  sourceRepos:
    - '*'  # Allow any Git repository (lab environment only)
  destinations:
    - namespace: '!kube-system'  # Explicit exclusion
      server: '*'
    - namespace: '*'            # Allow all other namespaces
      server: '*'
  clusterResourceWhitelist:
    - group: '*'               # Allow any cluster-scoped resources
      kind: '*'
```

**Security Trade-offs**:
- **Permissive source repos**: Allows rapid experimentation with external charts
- **Namespace protection**: Prevents accidental modification of system namespaces
- **Cluster resource access**: Enables testing of operators and custom resources
- **Lab environment justification**: Security relaxed for learning and development

### Production AppProject Example

For comparison, a production AppProject would be much more restrictive:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production-apps
  namespace: argocd
spec:
  description: 'Production applications with strict controls'
  sourceRepos:
    - 'https://github.com/goldentooth/helm-charts'
    - 'https://charts.bitnami.com/bitnami'
  destinations:
    - namespace: 'production-*'
      server: 'https://kubernetes.default.svc'
  clusterResourceWhitelist:
    - group: ''
      kind: 'Namespace'
    - group: 'rbac.authorization.k8s.io'
      kind: 'ClusterRole'
  namespaceResourceWhitelist:
    - group: 'apps'
      kind: 'Deployment'
    - group: ''
      kind: 'Service'
  roles:
    - name: 'developers'
      policies:
        - 'p, proj:production-apps:developers, applications, get, production-apps/*, allow'
        - 'p, proj:production-apps:developers, applications, sync, production-apps/*, allow'
```

## Application Configuration Patterns

### Sync Policy Configuration

The Application's sync policy defines automated behavior:

```yaml
syncPolicy:
  automated:
    prune: true      # Remove resources deleted from Git
    selfHeal: true   # Automatically fix configuration drift
  syncOptions:
    - Validate=true                    # Validate resources before applying
    - CreateNamespace=true             # Auto-create target namespaces
    - PrunePropagationPolicy=foreground # Wait for dependent resources
    - PruneLast=true                   # Delete applications last
    - RespectIgnoreDifferences=true    # Honor ignoreDifferences rules
    - ApplyOutOfSyncOnly=true         # Only apply changed resources
```

**Sync Policy Implications**:
- **Prune**: Ensures Git repository is single source of truth
- **Self-heal**: Prevents manual changes from persisting
- **Validation**: Catches configuration errors before deployment
- **Namespace creation**: Reduces manual setup for new applications

### Repository Structure for App-of-Apps

The incubator repository structure supports the app-of-apps pattern:

```
incubator/
├── README.md
├── applications/
│   ├── monitoring/
│   │   ├── prometheus.yaml
│   │   ├── grafana.yaml
│   │   └── alertmanager.yaml
│   ├── networking/
│   │   ├── metallb.yaml
│   │   ├── external-dns.yaml
│   │   └── cert-manager.yaml
│   └── storage/
│       ├── nfs-provisioner.yaml
│       ├── ceph-operator.yaml
│       └── seaweedfs.yaml
└── environments/
    ├── dev/
    ├── staging/
    └── production/
```

**Directory Organization Benefits**:
- **Logical grouping**: Applications organized by functional area
- **Environment separation**: Different configurations per environment
- **Clear ownership**: Teams can own specific directories
- **Selective deployment**: Enable/disable application groups easily

## Integration with ApplicationSets

### Migration Path from App-of-Apps

As applications mature, they can be migrated from the incubator to ApplicationSet management:

**Migration Steps**:
1. **Stabilize configuration**: Test thoroughly in incubator environment
2. **Create Helm chart**: Package application as reusable chart
3. **Add to gitops-repo**: Tag repository for ApplicationSet discovery
4. **Remove from incubator**: Delete Application from incubator repository
5. **Verify automation**: Confirm ApplicationSet creates new Application

**Example Migration**:
```yaml
# Before: Manual Application in incubator
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring-stack
  namespace: argocd
spec:
  project: incubator
  source:
    repoURL: 'https://github.com/goldentooth/monitoring'
    path: './manifests'

# After: Automatically generated by ApplicationSet
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
  ownerReferences:
  - apiVersion: argoproj.io/v1alpha1
    kind: ApplicationSet
    name: gitops-repo
spec:
  project: gitops-repo
  source:
    repoURL: 'https://github.com/goldentooth/monitoring'
    path: '.'
```

### ApplicationSet Template Advantages

**Consistent Configuration**:
- All applications get same sync policy
- Standardized labeling and annotations
- Uniform security settings across applications
- Reduced configuration drift between applications

**Template Parameters**:
```yaml
template:
  metadata:
    name: '{{repository}}'
    labels:
      environment: '{{environment}}'
      team: '{{team}}'
      gitops-managed: 'true'
  spec:
    project: '{{project}}'
    source:
      repoURL: '{{url}}'
      targetRevision: '{{branch}}'
      helm:
        valueFiles:
        - 'values-{{environment}}.yaml'
```

## Operational Workflows

### Development Workflow

**Incubator Development Process**:
1. **Create feature branch**: Develop new application in isolated branch
2. **Add Application manifest**: Define application in incubator repository
3. **Test deployment**: Verify application deploys correctly
4. **Iterate configuration**: Refine settings based on testing
5. **Merge to main**: Deploy to shared incubator environment
6. **Monitor and debug**: Observe application behavior and logs

### Production Promotion

**Graduation from Incubator**:
1. **Create dedicated repository**: Move application to own repository
2. **Package as Helm chart**: Standardize configuration management
3. **Add gitops-repo label**: Enable ApplicationSet discovery
4. **Configure environments**: Set up dev/staging/production values
5. **Test automation**: Verify ApplicationSet creates Application
6. **Remove from incubator**: Clean up experimental Application

### Monitoring and Observability

**Application Health Monitoring**:
```bash
# Check application sync status
kubectl -n argocd get applications

# View application details
argocd app get incubator

# Monitor sync operations
argocd app sync incubator --dry-run

# Check for configuration drift
argocd app diff incubator
```

**Common Issues and Troubleshooting**:
- **Sync failures**: Check resource validation and RBAC permissions
- **Resource conflicts**: Verify namespace isolation and resource naming
- **Git access issues**: Confirm repository permissions and authentication
- **Health check failures**: Review application health status and events

## Best Practices for GitOps

### Repository Management

**Separation of Concerns**:
- **Application code**: Business logic and container images
- **Configuration**: Kubernetes manifests and Helm values
- **Infrastructure**: Cluster setup and platform services
- **Policies**: Security rules and governance configurations

**Version Control Strategy**:
```
main branch    → Production environment
staging branch → Staging environment  
dev branch     → Development environment
feature/*      → Feature testing
```

### Security Considerations

**Credential Management**:
- Use Argo CD's credential templates for repository access
- Implement least-privilege access for Git repositories
- Rotate credentials regularly and audit access
- Consider using Git over SSH for enhanced security

**Resource Isolation**:
- Separate AppProjects for different security domains
- Use namespace-based isolation for applications
- Implement RBAC policies aligned with organizational structure
- Monitor cross-namespace resource access

This incubator approach provides a safe environment for experimenting with GitOps patterns while establishing the foundation for scalable, automated application management through ApplicationSets as the platform matures.
