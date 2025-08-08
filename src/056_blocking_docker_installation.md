# Blocking Docker Installation

## The Problem

I don't know why, and I'm too lazy to dig much into it, but if I install `docker` on any node in the Kubernetes cluster, this conflicts with containerd (`containerd.io`), which causes Kubernetes to shit blood and stop working on that node. Great.

To prevent this, I implemented a clusterwide ban on Docker. I'm recording the details here in case I need to do it again.

### Implementation

First, we removed Docker from nodes where it was already installed (like Allyrion):

```bash
# Stop and remove containers
goldentooth command_root allyrion "docker stop envoy && docker rm envoy"

# Remove all images
goldentooth command_root allyrion "docker images -q | xargs -r docker rmi -f"

# Stop and disable Docker
goldentooth command_root allyrion "systemctl stop docker && systemctl disable docker"
goldentooth command_root allyrion "systemctl stop docker.socket && systemctl disable docker.socket"

# Purge Docker packages
goldentooth command_root allyrion "apt-get purge -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin"
goldentooth command_root allyrion "apt-get autoremove -y"

# Clean up Docker directories
goldentooth command_root allyrion "rm -rf /var/lib/docker /etc/docker /var/run/docker.sock"
goldentooth command_root allyrion "rm -f /etc/apt/sources.list.d/docker.list /etc/apt/keyrings/docker.gpg"
```

### APT Preferences Configuration

Next, we added an APT preferences file to the `goldentooth.setup_security` role that blocks Docker packages from being installed:

```yaml
- name: 'Block Docker installation to prevent conflicts with Kubernetes containerd'
  ansible.builtin.copy:
    dest: '/etc/apt/preferences.d/block-docker'
    mode: '0644'
    owner: 'root'
    group: 'root'
    content: |
      # Block Docker installation to prevent conflicts with Kubernetes containerd
      # Docker packages can break the containerd installation used by Kubernetes
      # This preference file prevents accidental installation of Docker

      Package: docker-ce
      Pin: origin ""
      Pin-Priority: -1

      Package: docker-ce-cli
      Pin: origin ""
      Pin-Priority: -1

      Package: docker-ce-rootless-extras
      Pin: origin ""
      Pin-Priority: -1

      Package: docker-buildx-plugin
      Pin: origin ""
      Pin-Priority: -1

      Package: docker-compose-plugin
      Pin: origin ""
      Pin-Priority: -1

      Package: docker.io
      Pin: origin ""
      Pin-Priority: -1

      Package: docker-compose
      Pin: origin ""
      Pin-Priority: -1

      Package: docker-registry
      Pin: origin ""
      Pin-Priority: -1

      Package: docker-doc
      Pin: origin ""
      Pin-Priority: -1

      # Also block the older containerd.io package that comes with Docker
      # Kubernetes should use the standard containerd package instead
      Package: containerd.io
      Pin: origin ""
      Pin-Priority: -1
```

### Deployment

The configuration was deployed to all nodes using:

```bash
goldentooth configure_cluster
```

### Verification

We can verify that Docker is now blocked:

```bash
# Check Docker package policy
goldentooth command allyrion "apt-cache policy docker-ce"
# Output shows: Candidate: (none)

# Verify the preferences file exists
goldentooth command all "ls -la /etc/apt/preferences.d/block-docker"
```

## How APT Preferences Work

APT preferences allow you to control which versions of packages are installed. By setting a `Pin-Priority` of `-1`, we effectively tell APT to never install these packages, regardless of their availability in the configured repositories.

This is more robust than simply removing Docker repositories because:
1. It prevents installation from any source (including manual addition of repositories)
2. It provides clear documentation of why these packages are blocked
3. It's easily reversible if needed (just remove the preferences file)
