# Ceph Distributed Storage

With ZFS providing reliable local storage, I needed a distributed storage solution that could survive node failures and provide block storage for Kubernetes workloads. Enter Ceph: a battle-tested distributed storage system that, with some optimization, runs surprisingly well on our Raspberry Pi cluster.

## The Challenge

Goldentooth has three nodes (fenn, karstark, and lipps) with 450-500GB SSDs that we wanted to turn into resilient distributed storage. The challenge was making Ceph work within the constraints of Raspberry Pi hardware:

- **Limited RAM**: 8GB per node vs Ceph's default 4GB per OSD assumption
- **USB SSDs**: Not the fastest storage medium
- **ARM64 architecture**: Needed containerized deployment for easier management

## Architecture Decision

I chose the modern **cephadm** approach over traditional package-based installation:

- **Containerized**: Uses Podman to run Ceph services in containers
- **Orchestrated**: cephadm handles service placement and management
- **Minimal footprint**: Only 3 nodes for a production-ready cluster

### Service Layout
- **Monitors**: All 3 nodes (fenn, karstark, lipps) for quorum
- **Managers**: 2 nodes (fenn active, karstark standby)
- **OSDs**: 1 per node using the attached SSDs
- **No MDS/RGW**: CephFS and S3 gateway not needed initially

## Implementation

### Container Runtime Setup

Since Docker conflicts with Kubernetes' containerd, we used Podman:

```bash
goldentooth command_root fenn,karstark,lipps "apt install -y podman catatonit"
```

### Ceph Deployment

The `goldentooth.setup_ceph` Ansible role handles the entire deployment:

```yaml
# Install cephadm
- name: 'Install cephadm.'
  ansible.builtin.get_url:
    url: 'https://github.com/ceph/ceph/raw/pacific/src/cephadm/cephadm'
    dest: '/usr/local/bin/cephadm'
    mode: '0755'

# Bootstrap first node
- name: 'Bootstrap Ceph cluster.'
  ansible.builtin.command: >
    cephadm bootstrap --mon-ip {{ ipv4_address }}
    --cluster-network {{ network.infrastructure.cidr }}
    --ssh-user root
    --allow-overwrite
    --skip-pull
  when: inventory_hostname == groups['ceph'][0]
```

### Adding Storage

The next part was dealing with existing partitions on the SSDs. Ceph requires clean block devices:

```bash
# Clear partition tables
goldentooth command_root fenn,karstark,lipps "wipefs -a /dev/sda"

# Add OSDs
goldentooth command_root fenn "cephadm shell -- ceph orch daemon add osd fenn:/dev/sda"
goldentooth command_root fenn "cephadm shell -- ceph orch daemon add osd karstark:/dev/sda"
goldentooth command_root fenn "cephadm shell -- ceph orch daemon add osd lipps:/dev/sda"
```

### Storage Pools

I created two RBD (RADOS Block Device) pools optimized for our workloads:

```bash
# Kubernetes persistent volumes
ceph osd pool create kubernetes 32
ceph osd pool application enable kubernetes rbd
rbd pool init kubernetes

# General cluster storage
ceph osd pool create goldentooth-storage 16
ceph osd pool application enable goldentooth-storage rbd
rbd pool init goldentooth-storage
```

## Pi Optimizations

### Memory Tuning
Reduced memory allocations for Pi constraints:
- OSD memory target: 1GB (down from 4GB default)
- Monitor memory limit: 512MB
- Manager memory limit: 512MB

### Replication Strategy
Simple 3-node setup:
- Replication size: 2 (each object stored on 2 nodes)
- Minimum size: 1 (cluster stays operational with 1 copy)
- Can survive single node failure gracefully

### Monitoring Integration
Disabled Ceph's built-in node-exporter since we already have Prometheus node-exporter running on port 9100:

```bash
ceph orch rm node-exporter
```

## Final Status

After deployment, our cluster provides:

```
  cluster:
    id:     450c52f2-64cc-11f0-a634-d83add89ef61
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum fenn,karstark,lipps
    mgr: fenn.uhxzem(active), standbys: karstark.xeejkj
    osd: 3 osds: 3 up, 3 in

  data:
    pools:   3 pools, 49 pgs
    objects: 2 objects, 38 B
    usage:   871 MiB used, 1.4 TiB / 1.4 TiB avail
    pgs:     49 active+clean
```

**1.4 TiB** of distributed, fault-tolerant block storage ready for Kubernetes workloads.

## Command Interface

The deployment is integrated into the goldentooth CLI:

```bash
# Deploy Ceph cluster
goldentooth setup_ceph

# Check cluster status
goldentooth command_root fenn "cephadm shell -- ceph -s"

# Monitor health
goldentooth command_root fenn "cephadm shell -- ceph health detail"
```
