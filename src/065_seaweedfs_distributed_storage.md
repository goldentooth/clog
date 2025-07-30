# SeaweedFS Distributed Storage Implementation

With Ceph providing robust block storage for Kubernetes, Goldentooth needed an object storage solution optimized for file-based workloads. SeaweedFS emerged as the perfect complement: a simple, fast distributed file system that excels at handling large numbers of files with minimal operational overhead.

## The Architecture Decision

SeaweedFS follows a different philosophy from traditional distributed storage systems. Instead of complex replication schemes, it uses a simple master-volume architecture inspired by Google's Colossus and Facebook's Haystack:

- **Master servers**: Coordinate volume assignments and handle metadata
- **Volume servers**: Store actual file data in append-only volumes
- **No complex consensus**: Simple leader election and heartbeat-based failure detection

### Target Deployment

I chose a **dual-node setup** using fenn and karstark, each running both master and volume services:

- **Storage capacity**: ~1TB total (491GB + 515GB across dedicated SSDs)
- **Fault tolerance**: Each node can serve requests independently 
- **Architecture support**: Native ARM64 and x86_64 binaries
- **Version**: SeaweedFS 3.66 with automatic binary management

## Storage Foundation

The SeaweedFS deployment builds on the existing `goldentooth.bootstrap_seaweedfs` infrastructure:

### SSD Preparation
Each storage node gets a dedicated SSD mounted at `/mnt/seaweedfs-ssd/`:

```yaml
- name: Format SSD with ext4 filesystem
  ansible.builtin.filesystem:
    fstype: "{{ seaweedfs.filesystem_type }}"
    dev: "{{ seaweedfs.device }}"
    force: true

- name: Set proper ownership on SSD mount
  ansible.builtin.file:
    path: "{{ seaweedfs.mount_path }}"
    owner: "{{ seaweedfs.uid }}"
    group: "{{ seaweedfs.gid }}"
    mode: '0755'
    recurse: true
```

### Directory Structure
The bootstrap creates organized storage directories:
- `/mnt/seaweedfs-ssd/data/` - Volume server storage
- `/mnt/seaweedfs-ssd/master/` - Master server metadata
- `/mnt/seaweedfs-ssd/index/` - Volume indexing
- `/mnt/seaweedfs-ssd/filer/` - Future filer service data

## Service Implementation

The `goldentooth.setup_seaweedfs` role handles the complete service deployment:

### Binary Management
Cross-architecture support with automatic download:

```yaml
- name: Download SeaweedFS binary
  ansible.builtin.get_url:
    url: "https://github.com/seaweedfs/seaweedfs/releases/download/{{ seaweedfs.version }}/linux_arm64.tar.gz"
    dest: "/tmp/seaweedfs-{{ seaweedfs.version }}.tar.gz"
  when: ansible_architecture == "aarch64"

- name: Download SeaweedFS binary (x86_64)
  ansible.builtin.get_url:
    url: "https://github.com/seaweedfs/seaweedfs/releases/download/{{ seaweedfs.version }}/linux_amd64.tar.gz"
    dest: "/tmp/seaweedfs-{{ seaweedfs.version }}.tar.gz"
  when: ansible_architecture == "x86_64"
```

### Master Server Configuration
Each node runs a master server for metadata coordination:

```systemd
[Unit]
Description=SeaweedFS Master Server
After=network.target
Wants=network.target

[Service]
Type=simple
User=seaweedfs
Group=seaweedfs
ExecStart=/usr/local/bin/weed master \
    -port=9333 \
    -mdir=/mnt/seaweedfs-ssd/master \
    -ip=10.4.x.x
Restart=always
RestartSec=5s

# Security hardening
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/mnt/seaweedfs-ssd
```

### Volume Server Configuration
Volume servers handle the actual file storage:

```systemd
[Unit] 
Description=SeaweedFS Volume Server
After=network.target seaweedfs-master.service
Wants=network.target

[Service]
Type=simple
User=seaweedfs
Group=seaweedfs
ExecStart=/usr/local/bin/weed volume \
    -port=8080 \
    -dir=/mnt/seaweedfs-ssd/data \
    -max=64 \
    -mserver=10.4.x.x:9333 \
    -ip=10.4.x.x
Restart=always
RestartSec=5s

# Security hardening
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/mnt/seaweedfs-ssd
```

## Security Hardening

SeaweedFS services run with comprehensive systemd security constraints:

- **User isolation**: Dedicated `seaweedfs` user (UID/GID 985)
- **Filesystem protection**: `ProtectSystem=strict` with explicit write paths
- **Privilege containment**: `NoNewPrivileges=yes`
- **Process isolation**: `PrivateTmp=yes` and `ProtectHome=yes`

## Deployment Process

The deployment uses serial execution to ensure proper cluster formation:

```yaml
- name: Enable and start SeaweedFS services
  ansible.builtin.systemd:
    name: "{{ item }}"
    enabled: true
    state: started
    daemon_reload: true
  loop:
    - seaweedfs-master
    - seaweedfs-volume

- name: Wait for SeaweedFS master to be ready
  ansible.builtin.uri:
    url: "http://{{ ansible_default_ipv4.address }}:9333/cluster/status"
    method: GET
  until: master_health_check.status == 200
  retries: 10
  delay: 5
```

## Service Verification

Post-deployment health checks confirm proper operation:

### Master Server Status
```bash
curl http://10.4.8.46:9333/cluster/status
```

Returns cluster topology and master coordination status.

### Volume Server Status  
```bash
curl http://10.4.8.46:8080/status
```

Shows volume allocation and storage utilization.

### Volume Assignment Testing
```bash
curl -X POST http://10.4.8.46:9333/dir/assign
```

Demonstrates the master server's ability to assign write locations.

## Current Capabilities

The deployed SeaweedFS cluster provides:

- **Dual-master setup**: Both fenn and karstark can coordinate operations
- **Independent operation**: Each node can serve requests without clustering
- **REST API access**: Full HTTP interface for file operations
- **Systemd integration**: Auto-start, restart, and logging via journald
- **Storage capacity**: Nearly 1TB of distributed object storage

## Command Integration

SeaweedFS operations integrate with the goldentooth CLI:

```bash
# Deploy SeaweedFS cluster
goldentooth setup_seaweedfs

# Check service status
goldentooth command fenn,karstark "systemctl status seaweedfs-master seaweedfs-volume"

# View cluster status
goldentooth command fenn "curl -s http://localhost:9333/cluster/status | jq"

# Monitor storage utilization
goldentooth command fenn,karstark "df -h /mnt/seaweedfs-ssd"
```

## Future Enhancements

The current implementation establishes the foundation for advanced features:

- **True clustering**: Connect masters for high availability
- **Consul integration**: Service discovery and health monitoring  
- **Step-CA certificates**: TLS encryption for all communications
- **Filer service**: POSIX filesystem interface for applications
- **S3 gateway**: Amazon S3-compatible API endpoint
- **Prometheus metrics**: Integration with cluster monitoring

This SeaweedFS deployment provides Goldentooth with fast, simple object storage that complements the existing Ceph block storage, giving applications flexible storage options optimized for different workload patterns.