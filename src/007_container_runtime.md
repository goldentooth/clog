# Container Runtime

Kubernetes is a container orchestration platform and therefore requires some container runtime to be installed.

This is a simple step; [`containerd`](https://containerd.io) is well-supported, well-regarded, and I don't have any reason _not_ to use it.

I used Jeff Geerling's [Ansible role](https://github.com/geerlingguy/ansible-role-containerd) to install and configure `containerd` on my cluster; this is really the point at which some kind of IaC/configuration management system becomes something more than a polite suggestion 🙂

## Configuration Details

The containerd installation and configuration is managed through several key components:

### Ansible Role Configuration

The `geerlingguy.containerd` role is specified in my `requirements.yml` and configured with these critical variables in `group_vars/all/vars.yaml`:

```yaml
# geerlingguy.containerd configuration
containerd_package: 'containerd.io'
containerd_package_state: 'present' 
containerd_service_state: 'started'
containerd_service_enabled: true
containerd_config_cgroup_driver_systemd: true  # Critical for Kubernetes integration
```

### Runtime Integration with Kubernetes

The most important aspect of the containerd configuration is its integration with Kubernetes. The cluster explicitly configures the CRI socket path:

```yaml
kubernetes:
  cri_socket_path: 'unix:///var/run/containerd/containerd.sock'
```

This socket path is used throughout the kubeadm initialization and join processes, ensuring Kubernetes can communicate with the container runtime.

### Systemd Cgroup Driver

The configuration sets `SystemdCgroup = true` in the containerd configuration file (`/etc/containerd/config.toml`), which is essential because:

1. **Kubernetes 1.22+** requires systemd cgroup driver for kubelet
2. **Consistency**: Both kubelet and containerd must use the same cgroup driver
3. **Resource Management**: Enables proper CPU/memory limits enforcement

### Generated Configuration

The Ansible role generates a complete containerd configuration with these key settings:

```toml
# Runtime configuration
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup = true  # Critical for Kubernetes cgroup management

# Socket configuration  
[grpc]
address = "/run/containerd/containerd.sock"
```

### Installation Process

The Ansible role performs these steps:

1. **Repository Setup**: Adds Docker CE repository (containerd.io package source)
2. **Package Installation**: Installs `containerd.io` package  
3. **Default Config Generation**: Runs `containerd config default` to generate base config
4. **Systemd Cgroup Modification**: Patches config to set `SystemdCgroup = true`
5. **Service Management**: Enables and starts containerd service

### Architecture Support

The configuration automatically handles ARM64 architecture for the Raspberry Pi nodes through architecture detection in the Ansible variables, ensuring proper package selection for both ARM64 (Pi nodes) and AMD64 (x86 nodes).

### Troubleshooting Tools

The installation also provides `crictl` (Container Runtime Interface CLI) for debugging and inspecting containers directly at the runtime level, which proves invaluable when troubleshooting Kubernetes pod issues.

The container runtime installation is handled in my [`install_k8s_packages.yaml` playbook](https://github.com/goldentooth/cluster/blob/main/playbooks/install_k8s_packages.yaml), which is where we'll be spending some time in subsequent sections.
