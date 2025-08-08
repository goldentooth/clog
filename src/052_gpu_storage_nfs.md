# GPU Storage NFS Export

With the cluster expanded to include the Velaryon GPU node, a natural question emerged: how can the Raspberry Pi cluster nodes efficiently exchange data with the GPU node for machine learning workloads and other compute-intensive tasks?

The solution was to leverage Velaryon's secondary 1TB NVMe SSD and expose it to the entire cluster via NFS, creating a high-speed shared storage pool specifically for Pi-GPU data exchange.

## The Challenge

Velaryon came with two storage devices:
- Primary NVMe (nvme1n1): Linux system drive
- Secondary NVMe (nvme0n1): 1TB drive with old NTFS partitions from previous Windows installation

The goal was to repurpose this secondary drive as shared storage while maintaining architectural separation - the GPU node should provide storage services without becoming a structural component of the Pi cluster.

## Storage Architecture Decision

Rather than integrating Velaryon into the existing storage ecosystem (ZFS replication, Ceph distributed storage), I opted for a simpler approach:
- **Pure ext4**: Single partition consuming the entire 1TB drive
- **NFS export**: Simple, performant network filesystem
- **Subnet-wide access**: Available to all 10.4.x.x nodes

This keeps the GPU node loosely coupled while providing the needed functionality.

## Implementation

### Drive Preparation

First, I cleared the old NTFS partitions and created a fresh GPT layout:

```bash
# Clear existing partition table
sudo wipefs -af /dev/nvme0n1

# Create new GPT partition table and single partition
sudo parted /dev/nvme0n1 mklabel gpt
sudo parted /dev/nvme0n1 mkpart primary ext4 0% 100%

# Format as ext4
sudo mkfs.ext4 -L gpu-storage /dev/nvme0n1p1
```

The resulting filesystem has UUID `5bc38d5b-a7a4-426e-acdb-e5caf0a809d9` and is mounted persistently at `/mnt/gpu-storage`.

### NFS Server Configuration

Velaryon was configured as an NFS server with a single export:

```bash
# /etc/exports
/mnt/gpu-storage 10.4.0.0/20(rw,sync,no_root_squash,no_subtree_check)
```

This grants read-write access to the entire infrastructure subnet with synchronous writes for data integrity.

### Ansible Integration

Rather than manually configuring each node, I integrated the GPU storage into the existing Ansible automation:

**Inventory Updates** (`inventory/hosts`):
```yaml
nfs_server:
  hosts:
    allyrion:    # Existing NFS server
    velaryon:    # New GPU storage server
```

**Host Variables** (`inventory/host_vars/velaryon.yaml`):
```yaml
nfs_exports:
  - "/mnt/gpu-storage 10.4.0.0/20(rw,sync,no_root_squash,no_subtree_check)"
```

**Global Configuration** (`group_vars/all/vars.yaml`):
```yaml
nfs:
  mounts:
    primary:      # Existing allyrion NFS share
      share: "{{ hostvars[groups['nfs_server'] | first].ipv4_address }}:/mnt/usb1"
      mount: '/mnt/nfs'
      safe_name: 'mnt-nfs'
      type: 'nfs'
      options: {}
    gpu_storage:  # New GPU storage share
      share: "{{ hostvars['velaryon'].ipv4_address }}:/mnt/gpu-storage"
      mount: '/mnt/gpu-storage'
      safe_name: 'mnt-gpu\x2dstorage'  # Systemd unit name escaping
      type: 'nfs'
      options: {}
```

### Systemd Automount Configuration

The trickiest part was configuring systemd automount units. Systemd requires special character escaping for mount paths - the mount point `/mnt/gpu-storage` must use the unit name `mnt-gpu\x2dstorage` (where `\x2d` is the escaped dash).

**Mount Unit Template** (`templates/mount.j2`):
```systemd
[Unit]
Description=Mount {{ item.key }}

[Mount]
What={{ item.value.share }}
Where={{ item.value.mount }}
Type={{ item.value.type }}
{% if item.value.options -%}
Options={{ item.value.options | join(',') }}
{% else -%}
Options=defaults
{% endif %}

[Install]
WantedBy=default.target
```

**Automount Unit Template** (`templates/automount.j2`):
```systemd
[Unit]
Description=Automount {{ item.key }}
After=remote-fs-pre.target network-online.target network.target
Before=umount.target remote-fs.target

[Automount]
Where={{ item.value.mount }}
TimeoutIdleSec=60

[Install]
WantedBy=default.target
```

### Deployment Playbook

A new playbook `setup_gpu_storage.yaml` orchestrates the entire deployment:

```yaml
---
# Setup GPU storage on Velaryon with NFS export
- name: 'Setup Velaryon GPU storage and NFS export'
  hosts: 'velaryon'
  become: true
  tasks:
    - name: 'Ensure GPU storage mount point exists'
      ansible.builtin.file:
        path: '/mnt/gpu-storage'
        state: 'directory'
        owner: 'root'
        group: 'root'
        mode: '0755'

    - name: 'Check if GPU storage is mounted'
      ansible.builtin.command:
        cmd: 'mountpoint -q /mnt/gpu-storage'
      register: gpu_storage_mounted
      failed_when: false
      changed_when: false

    - name: 'Mount GPU storage if not already mounted'
      ansible.builtin.mount:
        src: 'UUID=5bc38d5b-a7a4-426e-acdb-e5caf0a809d9'
        path: '/mnt/gpu-storage'
        fstype: 'ext4'
        opts: 'defaults'
        state: 'mounted'
      when: gpu_storage_mounted.rc != 0

- name: 'Configure NFS exports on Velaryon'
  hosts: 'velaryon'
  become: true
  roles:
    - 'geerlingguy.nfs'

- name: 'Setup NFS mounts on all nodes'
  hosts: 'all'
  become: true
  roles:
    - 'goldentooth.setup_nfs_mounts'
```

## Usage

The GPU storage is now seamlessly integrated into the goldentooth CLI:

```bash
# Deploy/update GPU storage configuration
goldentooth setup_gpu_storage

# Enable automount on specific nodes
goldentooth command allyrion 'sudo systemctl enable --now mnt-gpu\x2dstorage.automount'

# Verify access (automounts on first access)
goldentooth command cargyll,bettley 'ls /mnt/gpu-storage/'
```

## Results

The implementation provides:

- **1TB shared storage** available cluster-wide at `/mnt/gpu-storage`
- **Automatic mounting** via systemd automount on directory access
- **Full Ansible automation** via the goldentooth CLI
- **Clean separation** between Pi cluster and GPU node architectures

Data written from any node is immediately visible across the cluster, enabling seamless Pi-GPU workflows for machine learning datasets, model artifacts, and computational results.
