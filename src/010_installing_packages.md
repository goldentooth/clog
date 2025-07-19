# Installing Packages

Now that we have functional access to the Kubernetes Apt package repository, we can install some important Kubernetes tools:

- `kubeadm` provides a straightforward way to setup and configure a Kubernetes cluster (API server, Controller Manager, DNS, etc). _Kubernetes the Hard Way_ basically does what `kubeadm` does. I use `kubeadm` because my goal is to go not necessarily _deeper_, but _farther_.
- `kubectl` is a CLI tool for administering a Kubernetes cluster; you can deploy applications, inspect resources, view logs, etc. As I'm studying for my CKA, I want to use `kubectl` for as much as possible.
- `kubelet` runs on each and every node in the cluster and ensures that pods are functioning as desired and takes steps to correct their behavior when it does not match the desired state.

## Package Installation Implementation

### Kubernetes Package Configuration

The package installation is managed through Ansible variables in `group_vars/all/vars.yaml`:

```yaml
kubernetes_version: '1.32'

kubernetes:
  apt_packages:
    - 'kubeadm'
    - 'kubectl' 
    - 'kubelet'
  apt_repo_url: "https://pkgs.k8s.io/core:/stable:/v{{ kubernetes_version }}/deb/"
```

This configuration:
- **Version management**: Uses Kubernetes 1.32 (latest stable at time of writing)
- **Repository pinning**: Uses version-specific repository for consistency
- **Package selection**: Core Kubernetes tools required for cluster operation

### Installation Process

The installation is handled by the `install_k8s_packages.yaml` playbook, which performs these steps:

**1. Container Runtime Setup**:
```yaml
- name: 'Setup `containerd`.'
  hosts: 'k8s_cluster'
  remote_user: 'root'
  roles:
    - { role: 'geerlingguy.containerd' }
```

This ensures containerd is installed and configured before Kubernetes packages.

**2. Package Installation**:
```yaml
- name: 'Install Kubernetes packages.'
  ansible.builtin.apt:
    name: "{{ kubernetes.apt_packages }}"
    state: 'present'
  notify:
    - 'Hold Kubernetes packages.'
    - 'Enable and restart kubelet service.'
```

**3. Package Hold Management**:
```yaml
- name: 'Hold Kubernetes packages.'
  ansible.builtin.dpkg_selections:
    name: "{{ package }}"
    selection: 'hold'
  loop: "{{ kubernetes.apt_packages }}"
```

This prevents accidental upgrades during regular system updates, ensuring cluster stability.

### Service Configuration

**kubelet Service Activation**:
```yaml
- name: 'Enable and restart kubelet service.'
  ansible.builtin.systemd_service:
    name: 'kubelet'
    state: 'restarted'
    enabled: true
    daemon_reload: true
```

Key features:
- **Auto-start**: Enables kubelet to start automatically on boot
- **Service restart**: Ensures kubelet starts with new configuration
- **Daemon reload**: Refreshes systemd to recognize any unit file changes

### Target Nodes

The installation targets the `k8s_cluster` inventory group, which includes:
- **Control plane nodes**: bettley, cargyll, dalt (3 nodes)
- **Worker nodes**: All remaining Raspberry Pi nodes + velaryon GPU node (10 nodes)

This ensures all cluster nodes have consistent Kubernetes tooling.

### Version Management Strategy

**Repository Strategy**:
- **Version-pinned repositories**: Uses `v1.32` specific repository
- **Package holds**: Prevents accidental upgrades via `dpkg --set-selections`
- **Coordinated updates**: Cluster-wide version management through Ansible

**Upgrade Process**:
1. Update `kubernetes_version` variable
2. Run `install_k8s_packages.yaml` playbook
3. Coordinate cluster upgrade using `kubeadm upgrade`
4. Update containerd and other runtime components as needed

### Integration with Container Runtime

The playbook ensures proper integration between Kubernetes and containerd:

**Runtime Configuration**:
- **CRI socket**: `/var/run/containerd/containerd.sock`
- **Cgroup driver**: systemd (required for Kubernetes 1.22+)
- **Image service**: containerd handles container image management

**Service Dependencies**:
- containerd must be running before kubelet starts
- kubelet configured to use containerd as container runtime
- Proper systemd service ordering ensures reliable startup

### Command Line Integration

The installation integrates with the goldentooth CLI:

```bash
# Install Kubernetes packages across cluster
goldentooth install_k8s_packages

# Uninstall if needed (cleanup)
goldentooth uninstall_k8s_packages
```

### Post-Installation Verification

After installation, you can verify the tools are properly installed:

```bash
# Check versions
goldentooth command k8s_cluster 'kubeadm version'
goldentooth command k8s_cluster 'kubectl version --client'
goldentooth command k8s_cluster 'kubelet --version'

# Verify package holds
goldentooth command k8s_cluster 'apt-mark showhold | grep kube'
```

Installing these tools is comparatively simple with the automated approach, just `sudo apt-get install -y kubeadm kubectl kubelet`, but the Ansible implementation adds important production considerations like version pinning, service management, and cluster-wide coordination that manual installation would miss.
