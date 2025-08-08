# SeaweedFS Distributed Storage Implementation

With Ceph providing robust block storage for Kubernetes, Goldentooth needed an object storage solution optimized for file-based workloads. SeaweedFS emerged as the perfect complement: a simple, fast distributed file system that excels at handling large numbers of files with minimal operational overhead.

## The Architecture Decision

SeaweedFS follows a different philosophy from traditional distributed storage systems. Instead of complex replication schemes, it uses a simple master-volume architecture inspired by Google's Colossus and Facebook's Haystack:

- **Master servers**: Coordinate volume assignments with HashiCorp Raft consensus
- **Volume servers**: Store actual file data in append-only volumes
- **HA consensus**: Raft-based leadership election with automatic failover

### Target Deployment

I implemented a **high availability cluster** using fenn and karstark with true HA clustering:

- **Storage capacity**: ~1TB total (491GB + 515GB across dedicated SSDs)
- **Fault tolerance**: Automatic failover with zero-downtime leadership transitions
- **Consensus protocol**: HashiCorp Raft for distributed coordination
- **Architecture support**: Native ARM64 and x86_64 binaries
- **Version**: SeaweedFS 3.66 with HA clustering capabilities

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

### High Availability Master Configuration
Each node runs a master server with HashiCorp Raft consensus for true HA clustering:

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
    -ip=10.4.x.x \
    -peers=fenn:9333,karstark:9333 \
    -raftHashicorp=true \
    -defaultReplication=001 \
    -volumeSizeLimitMB=1024
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
Volume servers automatically track the current cluster leader:

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
    -mserver=fenn:9333,karstark:9333 \
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

### HA Cluster Status
```bash
curl http://fenn:9333/cluster/status
```

Returns cluster topology, current leader, and peer status.

### Leadership Monitoring
```bash
# Watch leadership changes (healthy flapping every 3 seconds)
watch -n 1 'curl -s http://fenn:9333/cluster/status | jq .Leader'
```

### Volume Server Status
```bash
curl http://fenn:8080/status
```

Shows volume allocation and current master server connections.

### Volume Assignment Testing
```bash
curl -X POST http://fenn:9333/dir/assign
```

Demonstrates automatic request routing to the current cluster leader.

## High Availability Cluster Status

The SeaweedFS cluster now operates as a true HA system:

- **Raft consensus**: HashiCorp Raft manages leadership election and state replication
- **Automatic failover**: Zero-downtime master transitions when nodes fail
- **Leadership rotation**: Healthy 3-second leadership cycling for load balancing
- **Cluster awareness**: Volume servers automatically follow leadership changes
- **Fault tolerance**: Cluster recovers gracefully from network partitions
- **Storage capacity**: Nearly 1TB with redundancy and automatic replication

## Command Integration

SeaweedFS operations integrate with the goldentooth CLI:

```bash
# Deploy SeaweedFS cluster
goldentooth setup_seaweedfs

# Check HA cluster status
goldentooth command fenn,karstark "systemctl status seaweedfs-master seaweedfs-volume"

# View cluster leadership and peers
goldentooth command fenn "curl -s http://localhost:9333/cluster/status | jq"

# Monitor leadership changes
goldentooth command fenn "watch -n 1 'curl -s http://localhost:9333/cluster/status | jq .Leader'"

# Monitor storage utilization
goldentooth command fenn,karstark "df -h /mnt/seaweedfs-ssd"
```
