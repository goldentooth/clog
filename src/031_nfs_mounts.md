# NFS Mounts

Now that Kubernetes is _kinda_ squared away, I'm going to set up NFS mounts on the cluster nodes.

For the sake of simplicity, I'll just set up the mounts on every node, including the load balancer (which is currently exporting the share).

## Implementation Architecture

### Systemd-Based Mounting

Rather than using traditional `/etc/fstab` entries, I implemented NFS mounting using systemd mount and automount units. This approach provides several advantages:

- **Dynamic mounting**: Automount units mount filesystems on-demand
- **Service management**: Standard systemd service lifecycle management
- **Dependency handling**: Proper ordering with network services
- **Logging**: Integration with systemd journal for troubleshooting

### Global Configuration

The NFS mount configuration is defined in `group_vars/all/vars.yaml`:

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
- **Dynamically determines NFS server**: Uses first host in `nfs_server` group (allyrion)
- **IP-based addressing**: Uses `10.4.0.10:/mnt/usb1` for reliable connectivity
- **Standardized mount point**: All nodes mount at `/mnt/nfs`
- **Safe naming**: Provides `mnt-nfs` for systemd unit names

## Systemd Template Implementation

### Mount Unit Template

The mount service template (`templates/mount.j2`) creates individual systemd mount units:

```systemd
[Unit]
Description=Mount {{ item.key }}

[Mount]
What={{ item.value.share }}
Where={{ item.value.mount }}
Type={{ item.value.type }}
Options={{ item.value.options | join(',') }}

[Install]
WantedBy=default.target
```

This generates a unit file at `/etc/systemd/system/mnt-nfs.mount` with:
- **What**: `10.4.0.10:/mnt/usb1` (NFS export path)
- **Where**: `/mnt/nfs` (local mount point)
- **Type**: `nfs` (filesystem type)
- **Options**: Default NFS mount options

### Automount Unit Template

The automount template (`templates/automount.j2`) provides on-demand mounting:

```systemd
[Unit]
Description=Automount {{ item.key }}
After=remote-fs-pre.target network-online.target network.target
Before=umount.target remote-fs.target

[Automount]
Where={{ item.value.mount }}

[Install]
WantedBy=default.target
```

Key features:
- **Network dependencies**: Waits for network availability before attempting mounts
- **Lazy mounting**: Only mounts when the path is accessed
- **Proper ordering**: Correctly sequences with system startup and shutdown

## Deployment Process

### Ansible Role Implementation

The `goldentooth.setup_nfs_mounts` role handles the complete deployment:

```yaml
- name: 'Generate mount unit for {{ item.key }}.'
  ansible.builtin.template:
    src: 'mount.j2'
    dest: "/etc/systemd/system/{{ item.value.safe_name }}.mount"
    mode: '0644'
  loop: "{{ nfs.mounts | dict2items }}"
  notify: 'reload systemd'

- name: 'Generate automount unit for {{ item.key }}.'
  ansible.builtin.template:
    src: 'automount.j2'
    dest: "/etc/systemd/system/{{ item.value.safe_name }}.automount"
    mode: '0644'
  loop: "{{ nfs.mounts | dict2items }}"
  notify: 'reload systemd'
```

### Service Management

The role ensures proper service lifecycle:

```yaml
- name: 'Enable and start automount services.'
  ansible.builtin.systemd:
    name: "{{ item.value.safe_name }}.automount"
    enabled: true
    state: started
    daemon_reload: true
  loop: "{{ nfs.mounts | dict2items }}"
```

## Network Integration

### Client Targeting

The NFS mounts are deployed across the entire cluster:

**Target Hosts**: All cluster nodes (`hosts: 'all'`)
- **12 Raspberry Pi nodes**: allyrion, bettley, cargyll, dalt, erenford, fenn, gardener, harlton, inchfield, jast, karstark, lipps
- **1 x86 GPU node**: velaryon

**Including NFS Server**: Even allyrion (the NFS server) mounts its own export, providing:
- **Consistent access patterns**: Same path (`/mnt/nfs`) on all nodes
- **Testing capability**: Server can verify export functionality
- **Simplified administration**: Uniform management across cluster

### Network Configuration

**Infrastructure Network**: All communication occurs within the trusted `10.4.0.0/20` CIDR
**NFS Protocol**: Standard NFSv3/v4 with default options
**Firewall**: No additional firewall rules needed within cluster network

## Directory Structure and Permissions

### Mount Point Creation

```yaml
- name: 'Ensure mount directories exist.'
  ansible.builtin.file:
    path: "{{ item.value.mount }}"
    state: directory
    mode: '0755'
  loop: "{{ nfs.mounts | dict2items }}"
```

### Shared Directory Usage

The NFS mount serves multiple cluster functions:

**Slurm Integration**:
```yaml
slurm_nfs_base_path: "{{ nfs.mounts.primary.mount }}/slurm"
```

**Common Patterns**:
- `/mnt/nfs/slurm/` - HPC job shared storage
- `/mnt/nfs/shared/` - General cluster shared data
- `/mnt/nfs/config/` - Configuration file distribution

## Command Line Integration

### goldentooth CLI Commands

```bash
# Configure NFS mounts on all nodes
goldentooth setup_nfs_mounts

# Verify mount status
goldentooth command all 'systemctl status mnt-nfs.automount'
goldentooth command all 'df -h /mnt/nfs'

# Test shared storage
goldentooth command allyrion 'echo "test" > /mnt/nfs/test.txt'
goldentooth command bettley 'cat /mnt/nfs/test.txt'
```

## Troubleshooting and Verification

### Service Status Verification

```bash
# Check automount service status
systemctl status mnt-nfs.automount

# Check mount service status (after access)
systemctl status mnt-nfs.mount

# View mount information
mount | grep nfs
df -h /mnt/nfs
```

### Common Issues and Solutions

**Network Dependencies**: The automount units properly wait for network availability through `After=network-online.target`

**Permission Issues**: The NFS export uses `no_root_squash`, allowing proper root access from clients

**Mount Persistence**: Automount units ensure mounts survive reboots and network interruptions

## Security Considerations

### Trust Model

**Internal Network Security**: Security relies on the trusted cluster network boundary
**No User Authentication**: Uses IP-based access control rather than user credentials
**Root Access**: `no_root_squash` on server allows administrative operations

### Future Enhancements

The current implementation could be enhanced with:
- **Kerberos authentication** for user-based security
- **Network policies** for additional access control
- **Encryption in transit** for sensitive data protection

## Integration with Storage Evolution

*Note: This NFS mounting system provides the foundation for shared storage. As documented in Chapter 050, the cluster later evolves to include ZFS-based storage with replication, while maintaining compatibility with these NFS mount patterns.*

This in itself wasn't too complicated, but I created two template files (one for a .mount service, another for a .automount service), fought with the variables for a bit, and it seems to work. The result is robust, cluster-wide shared storage accessible at `/mnt/nfs` on every node.
