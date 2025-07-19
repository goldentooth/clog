# Installing Helm

I have a lot of ambitions for this cluster, but after some deliberation, the thing I most want to do right now is deploy something _to_ Kubernetes.

So I'll be starting out by installing Argo CD, and I'll do that... soon. In the next chapter. I decided to install Argo CD via Helm, since I expect that Helm will be useful in other situations as well, e.g. trying out applications before I commit (no pun intended) to bringing them into GitOps.

So I created a [playbook](https://github.com/goldentooth/cluster/blob/main/playbooks/install_helm.yaml) and [role](https://github.com/goldentooth/cluster/tree/main/roles/goldentooth.install_helm) to cover installing Helm.

## Installation Implementation

### Package Repository Approach

Rather than downloading binaries manually, I chose to use the official Helm APT repository for a more maintainable installation. The Ansible role adds the repository using the modern `deb822_repository` format:

```yaml
- name: 'Add Helm package repository.'
  ansible.builtin.deb822_repository:
    name: 'helm'
    types: ['deb']
    uris: ['https://baltocdn.com/helm/stable/debian/']
    suites: ['all']
    components: ['main']
    architectures: ['arm64']
    signed_by: 'https://baltocdn.com/helm/signing.asc'
```

This approach provides several benefits:
- **Automatic updates**: Using `state: 'latest'` ensures we get the most recent Helm version
- **Security**: Uses the official Helm signing key for package verification
- **Architecture support**: Properly configured for ARM64 architecture on Raspberry Pi nodes
- **Maintainability**: Standard package management simplifies updates and removes manual binary management

### Installation Scope

Helm is installed only on the **Kubernetes control plane nodes** (`k8s_control_plane` group). This is sufficient because:

1. **Post-Tiller Architecture**: Modern Helm (v3+) doesn't require a server-side component
2. **Client-only Tool**: Helm operates entirely as a client-side tool that communicates with the Kubernetes API
3. **Administrative Access**: Control plane nodes are where cluster administration typically occurs
4. **Resource Efficiency**: No need to install on every worker node

### Integration with Cluster Architecture

**Kubernetes Integration**:
The installation leverages the existing `kubernetes.core` Ansible collection, ensuring proper integration with the cluster's Kubernetes components. The role depends on:

- Existing cluster RBAC configurations
- Kubernetes API server access from control plane nodes
- Standard kubeconfig files for authentication

**GitOps Integration**:
Helm serves as a crucial component for the GitOps workflow, particularly for Argo CD installation. The integration follows this pattern:

```yaml
- name: 'Add Argo Helm chart repository.'
  kubernetes.core.helm_repository:
    name: 'argo'
    repo_url: "{{ argo_cd.chart_repo_url }}"

- name: 'Install Argo CD from Helm chart.'
  kubernetes.core.helm:
    atomic: false
    chart_ref: 'argo/argo-cd'
    chart_version: "{{ argo_cd.chart_version }}"
    create_namespace: true
    release_name: 'argocd'
    release_namespace: "{{ argo_cd.namespace }}"
```

### Security Considerations

The installation follows security best practices:

- **Signed Packages**: Uses official Helm signing key for package verification
- **Trusted Repository**: Sources packages directly from Helm's CDN
- **No Custom RBAC**: Relies on existing Kubernetes cluster RBAC rather than creating additional permissions
- **System-level Installation**: Installed as root for proper system integration

### Command Line Integration

The installation integrates seamlessly with the goldentooth CLI:

```bash
goldentooth install_helm
```

This command maps directly to the Ansible playbook execution, maintaining consistency with the cluster's unified management interface.

### Version Management Strategy

The configuration uses a `state: 'latest'` strategy, which means:

- **Automatic Updates**: Each playbook run ensures the latest Helm version is installed
- **Application-level Pinning**: Specific chart versions are managed at the application level (e.g., Argo CD chart version 7.1.5)
- **Simplified Maintenance**: No need to manually track Helm version updates

### High Availability Considerations

By installing Helm on all control plane nodes, the configuration provides:

- **Redundancy**: Any control plane node can perform Helm operations
- **Administrative Flexibility**: Cluster administrators can use any control plane node
- **Disaster Recovery**: Helm operations can continue even if individual control plane nodes fail

Fortunately, this is fairly simple to install and trivial to configure, which is not something I can say for Argo CD 🙂
