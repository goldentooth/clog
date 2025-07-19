# NFS Exports

Just kidding, I'm going to set up a USB thumb drive and NFS exports on Allyrion (my load balancer node).

The thumbdrive is just a Sandisk 64GB. Should be enough to do some fun stuff. `fdisk` it (hey, I remember the commands!), `mkfs.ext4` it, get the UUID, add it to `/etc/fstab` (not "f-stab", "fs-tab"), and we have a bright shiny new volume.

## NFS Server Implementation

NFS isn't hard to set up, but I'm going to use Jeff's [ansible-role-nfs](https://github.com/geerlingguy/ansible-role-nfs) for consistency and maintainability.

The implementation consists of two main components:

### Server Configuration

The NFS server setup is managed through the `setup_nfs_exports.yaml` playbook, which performs these operations:

1. **Install NFS utilities on all nodes**:
```yaml
- name: 'Install NFS utilities.'
  hosts: 'all'
  remote_user: 'root'
  tasks:
    - name: 'Ensure NFS utilities are installed.'
      ansible.builtin.apt:
        name:
          - nfs-common
        state: present
```

2. **Configure NFS server on allyrion**:
```yaml
- name: 'Setup NFS exports.'
  hosts: 'nfs'
  remote_user: 'root'
  roles:
    - { role: 'geerlingguy.nfs' }
```

### Export Configuration

The NFS export is configured through host variables in `inventory/host_vars/allyrion.yaml`:

```yaml
nfs_exports:
- "/mnt/usb1 *(rw,sync,no_root_squash,no_subtree_check)"
```

This export configuration provides:

- **Path**: `/mnt/usb1` - The USB thumb drive mount point
- **Access**: `*` - Allow access from any host within the cluster network
- **Permissions**: `rw` - Read-write access for all clients
- **Sync Policy**: `sync` - Synchronous writes (safer but slower than async)
- **Root Mapping**: `no_root_squash` - Allow root user from clients to maintain root privileges
- **Performance**: `no_subtree_check` - Disable subtree checking for better performance

### Network Integration

The NFS server integrates with the cluster's network architecture:

**Server Information**:
- **Host**: allyrion (10.4.0.10)
- **Role**: Dual-purpose load balancer and NFS server
- **Network**: Infrastructure CIDR `10.4.0.0/20`

**Global NFS Configuration** (in `group_vars/all/vars.yaml`):
```yaml
nfs:
  server: "{{ groups['nfs_server'] | first}}"
  mounts:
    primary:
      share: "{{ hostvars[groups['nfs_server'] | first].ipv4_address }}:/mnt/usb1"
      mount: '/mnt/nfs'
      safe_name: 'mnt-nfs'
      type: 'nfs'
      options: {}
```

This configuration:
- Dynamically determines the NFS server from the `nfs_server` inventory group
- Uses the server's IP address for robust connectivity
- Standardizes the client mount point as `/mnt/nfs`
- Provides a safe filesystem name for systemd units

### Security Considerations

**Internal Network Trust Model**:
The NFS configuration uses a simplified security model appropriate for an internal cluster:

- **Open Access**: The `*` wildcard allows any host to mount the share
- **No Kerberos**: Uses IP-based authentication rather than user-based
- **Root Access**: `no_root_squash` enables administrative operations from clients
- **Network Boundary**: Security relies on the trusted internal network (`10.4.0.0/20`)

### Storage Infrastructure

**Physical Storage**:
- **Device**: SanDisk 64GB USB thumb drive
- **Filesystem**: ext4 for reliability and broad compatibility
- **Mount Point**: `/mnt/usb1` 
- **Persistence**: Configured in `/etc/fstab` using UUID for reliability

**Performance Characteristics**:
- **Capacity**: 64GB available storage
- **Access Pattern**: Shared read-write across 13 cluster nodes
- **Use Cases**: Configuration files, shared data, cluster coordination

### Verification and Testing

The NFS export can be verified using standard tools:

```bash
$ showmount -e allyrion
Exports list on allyrion:
/mnt/usb1                           *
```

This output confirms:
- The export is properly configured and accessible
- The path `/mnt/usb1` is being served
- Access is open to all hosts (`*`)

### Command Line Integration

The NFS setup integrates with the goldentooth CLI for consistent cluster management:

```bash
# Configure NFS server
goldentooth setup_nfs_exports

# Configure client mounts (covered in Chapter 031)
goldentooth setup_nfs_mounts

# Verify exports
goldentooth command allyrion 'showmount -e allyrion'
```

### Future Evolution

*Note: This represents the initial NFS implementation. The cluster later evolves to include more sophisticated storage with ZFS pools and replication (documented in Chapter 050), while maintaining compatibility with this foundational NFS export.*

We'll return to this later and find out if it _actually_ works when we configure the client mounts in the next section.
