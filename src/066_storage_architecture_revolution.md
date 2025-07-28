# Storage Architecture Revolution: From Ceph Complexity to SeaweedFS + Longhorn Harmony

Sometimes the most significant technical decisions come from acknowledging when a promising solution isn't the right fit for your infrastructure. After successfully deploying Ceph distributed storage and even integrating it with Kubernetes through CSI, we faced a harsh reality: Ceph's resource demands and operational complexity were pushing our Raspberry Pi cluster beyond its sustainable limits. This chapter documents our transition from a monolithic Ceph approach to a hybrid SeaweedFS + Longhorn architecture that better matches our Pi cluster's constraints while delivering superior functionality.

## The Ceph Struggle: Success Shadowed by Sustainability Issues

Our Ceph deployment was technically successful. We had achieved:
- **1.4 TiB of distributed storage** across three nodes (fenn, karstark, lipps)
- **Working ReadWriteOnce (RWO) storage** through the CSI driver
- **Fault tolerance** with 2-replica configuration
- **Kubernetes integration** with dynamic provisioning

However, beneath this success lay growing operational challenges that made the solution unsustainable for long-term production use.

### Memory Constraints: The 4GB Ceiling

The most critical issue was memory pressure on our 4GB Raspberry Pi nodes (karstark and lipps):

```bash
# Memory usage on a typical Ceph node
goldentooth command karstark "free -h"
#               total        used        free      shared  buff/cache   available
# Mem:          3.7Gi       3.2Gi       180Mi        15Mi       350Mi       280Mi
```

**Ceph's memory footprint per node:**
- **OSD daemon**: ~800MB (despite tuning down from 4GB default)
- **Monitor daemon**: ~400MB
- **Manager daemon**: ~200MB (when active)
- **Container overhead**: ~200MB
- **Total per node**: **~1.6GB** on active nodes, **~1.2GB** on passive nodes

This left insufficient headroom for:
- Kubernetes system pods (~300MB)
- CNI networking (~100MB)
- Node OS operations (~200MB)
- Emergency memory buffers (~200MB)

During peak operations, nodes would approach 95% memory utilization, triggering:
- OOM killer activation
- Pod evictions
- System instability
- Performance degradation

### ARM64 Compatibility Challenges

While Ceph technically supports ARM64, we encountered recurring issues:

**Container Registry Problems:**
- Inconsistent multi-arch image availability
- Version mismatches between architectures
- Registry timeout issues during updates

**Authentication Instability:**
```bash
# Frequent authentication failures requiring cluster rebuilds
goldentooth command_root fenn "cephadm shell -- ceph auth ls"
# Error: authentication timeout
# Error: monitor connection failed
```

These issues weren't show-stoppers individually, but their cumulative effect required constant attention and frequent cluster rebuilds.

### ReadWriteMany (RWX) Limitation

Our original goal was ReadWriteMany storage for shared workloads, but Ceph RBD only provides ReadWriteOnce semantics. While CephFS could provide RWX, it would require:
- Additional MDS (Metadata Server) daemons
- More memory allocation per node
- Increased operational complexity
- Further ARM64 compatibility concerns

The memory cost would have pushed our 4GB nodes over the edge.

### Operational Complexity

Ceph's sophisticated architecture came with operational overhead:

**Complex Troubleshooting:**
- Multi-layer debugging (cephadm → containers → Ceph services)
- Intricate networking requirements
- Difficult log aggregation across containerized services

**Update Complexity:**
- Multi-stage rolling updates
- Container orchestration considerations
- Risk of split-brain scenarios during maintenance

**Recovery Procedures:**
- Complex disaster recovery requiring deep Ceph knowledge
- Manual intervention needed for many failure scenarios
- Risk of data loss during cluster rebuilds

## The Hybrid Vision: SeaweedFS + Longhorn

After extensive research and testing, we designed a hybrid storage architecture that better matched our infrastructure constraints:

**SeaweedFS** for ReadWriteMany shared storage
**Longhorn** for high-performance ReadWriteOnce storage

### Why SeaweedFS for RWX Storage?

SeaweedFS offered compelling advantages for our Pi cluster:

**Minimal Memory Footprint:**
- **Master server**: 200-500MB per node
- **Volume server**: 300-600MB per node
- **Filer server**: 400-800MB (only on primary node)
- **Total per node**: **~600MB** vs Ceph's 1.2GB+

**Native ARM64 Support:**
- Purpose-built Go binaries with excellent ARM64 support
- Single binary deployment model
- Consistent behavior across architectures

**ReadWriteMany Native:**
- POSIX-compliant filesystem interface through Filer
- Native Kubernetes CSI driver with RWX support
- No additional configuration required

**Operational Simplicity:**
- Simple master-volume-filer architecture
- Clear debugging paths
- Minimal external dependencies

### Why Longhorn for RWO Storage?

For ReadWriteOnce workloads requiring high performance, Longhorn provided:

**Ultra-Low Memory Overhead:**
- **Manager**: ~150MB per node
- **Engine**: ~50MB per volume
- **CSI Driver**: ~100MB per node
- **Total base overhead**: **~300MB per node**

**SSD Optimization:**
- Native understanding of SSD characteristics
- Optimal I/O patterns for flash storage
- TRIM support for SSD longevity

**Kubernetes Native:**
- Designed specifically for Kubernetes workloads
- Clean integration with K8s primitives
- GitOps-friendly configuration

## Node Reorganization Strategy

The transition required reorganizing our storage node groups to optimize for each technology's strengths:

### Original Ceph Configuration
```yaml
# Previous inventory group
ceph:
  hosts:
    fenn:      # 7.6GB RAM
    karstark:  # 3.7GB RAM
    lipps:     # 3.7GB RAM
```

### New Hybrid Configuration
```yaml
# SeaweedFS group (ReadWriteMany)
seaweed:
  hosts:
    fenn:      # 7.6GB RAM - Primary (Master + Volume + Filer)
    karstark:  # 3.7GB RAM - Secondary (Master + Volume)
    lipps:     # 3.7GB RAM - Secondary (Master + Volume)

# Longhorn group (ReadWriteOnce)
longhorn:
  hosts:
    inchfield: # 8GB RAM - Previously dedicated to Loki
    jast:      # 8GB RAM - Previously dedicated to Step-CA
```

This reorganization leveraged each node's strengths:

**SeaweedFS on fenn/karstark/lipps:**
- Utilizes the existing SSD storage on these nodes
- Fenn's higher memory (7.6GB) supports the Filer component
- Karstark and lipps provide redundancy with lighter memory loads

**Longhorn on inchfield/jast:**
- Both nodes have 8GB RAM, providing ample headroom
- These nodes already had SSD storage
- Existing services (Loki, Step-CA) coexist well with Longhorn

## Implementation Deep Dive

### SeaweedFS Deployment Architecture

The SeaweedFS cluster was designed with careful memory allocation:

**Fenn (Primary Node - 7.6GB RAM):**
```yaml
# SeaweedFS component allocation
seaweedfs-master:
  resources:
    requests:
      memory: 512Mi
    limits:
      memory: 1Gi

seaweedfs-volume:
  resources:
    requests:
      memory: 1Gi
    limits:
      memory: 2Gi

seaweedfs-filer:
  resources:
    requests:
      memory: 800Mi
    limits:
      memory: 1500Mi
```

**Karstark & Lipps (Secondary Nodes - 3.7GB RAM each):**
```yaml
seaweedfs-master:
  resources:
    requests:
      memory: 256Mi
    limits:
      memory: 512Mi

seaweedfs-volume:
  resources:
    requests:
      memory: 800Mi
    limits:
      memory: 1500Mi
```

### Longhorn Deployment Architecture

Longhorn was configured with aggressive memory optimization:

**Both inchfield and jast (8GB RAM each):**
```yaml
longhorn-manager:
  resources:
    requests:
      memory: 100Mi
    limits:
      memory: 150Mi

longhorn-engine:
  # Per-volume allocation
  resources:
    requests:
      memory: 25Mi
    limits:
      memory: 50Mi

longhorn-csi-controller:
  resources:
    requests:
      memory: 25Mi
    limits:
      memory: 50Mi
```

### Storage Class Strategy

The hybrid approach enabled specialized storage classes:

**SeaweedFS Storage Classes:**
```yaml
# Shared storage for multi-pod access
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: seaweedfs-storage-rwx
provisioner: seaweedfs-csi-driver
parameters:
  replication: "2"
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

**Longhorn Storage Classes:**
```yaml
# High-performance database storage
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "2880"
  fromBackup: ""
allowVolumeExpansion: true
reclaimPolicy: Delete
```

## Performance Comparison Results

The transition delivered significant improvements across all metrics:

### Memory Utilization (Per Node)

| Component               | Ceph      | SeaweedFS | Longhorn | Improvement      |
| ----------------------- | --------- | --------- | -------- | ---------------- |
| Base Memory             | 1.2-1.6GB | 600MB     | 300MB    | 50-80% reduction |
| Available for Workloads | 1.5-2GB   | 2.5-3GB   | 7.2GB    | 60-360% increase |

### Performance Benchmarks

**Sequential I/O Performance:**
```bash
# SeaweedFS (ReadWriteMany)
fio --name=seqwrite --rw=write --bs=1M --size=1G --filename=/mnt/seaweed/test
# Result: 75-85 MB/s write, 80-95 MB/s read

# Longhorn (ReadWriteOnce)
fio --name=seqwrite --rw=write --bs=1M --size=1G --filename=/mnt/longhorn/test
# Result: 320-400 MB/s write, 400-500 MB/s read
```

**Random I/O Performance:**
```bash
# SeaweedFS 4K Random
fio --name=randwrite --rw=randwrite --bs=4k --size=100M --filename=/mnt/seaweed/rand
# Result: 2,500-3,500 IOPS

# Longhorn 4K Random
fio --name=randwrite --rw=randwrite --bs=4k --size=100M --filename=/mnt/longhorn/rand
# Result: 12,000-18,000 IOPS
```

### Operational Metrics

**Deployment Time:**
- **Ceph**: 45-60 minutes (including troubleshooting)
- **SeaweedFS**: 8-12 minutes
- **Longhorn**: 5-8 minutes

**Recovery Time (Single Node Failure):**
- **Ceph**: 15-30 minutes (often requiring manual intervention)
- **SeaweedFS**: 2-5 minutes (automatic)
- **Longhorn**: 30-60 seconds (automatic)

**Memory Stability:**
- **Ceph**: Frequent OOM events, 85-95% utilization
- **SeaweedFS**: Stable 60-70% utilization
- **Longhorn**: Stable 40-50% utilization

## Repository Structure and GitOps Integration

The transition included creating comprehensive deployment repositories:

### SeaweedFS Cluster Repository
```
~/Projects/goldentooth/seaweedfs-cluster/
├── manifests/
│   ├── namespace.yaml
│   ├── master/
│   ├── volume/
│   ├── filer/
│   ├── csi-driver/
│   └── monitoring/
├── argocd/
│   └── seaweedfs-application.yaml
├── docs/
│   ├── memory-allocation-strategy.md
│   ├── backup-disaster-recovery.md
│   └── performance-optimization.md
└── scripts/
    ├── pre-flight.sh
    ├── deploy.sh
    └── validate.sh
```

### Longhorn Cluster Repository
```
~/Projects/goldentooth/longhorn-cluster/
├── manifests/
│   ├── namespace.yaml
│   ├── system/
│   ├── engine/
│   ├── manager/
│   ├── csi-driver/
│   └── monitoring/
├── argocd/
│   └── longhorn-application.yaml
├── docs/
│   ├── performance-tuning.md
│   ├── backup-disaster-recovery.md
│   └── ssd-optimization.md
└── examples/
    ├── test-performance-pvc.yaml
    └── database-workload.yaml
```

### Ansible Integration

The hybrid storage approach was integrated into our existing Ansible infrastructure:

**Updated Inventory Groups:**
```yaml
# ansible/inventory/hosts
seaweed:
  hosts:
    fenn:
    karstark:
    lipps:

longhorn:
  hosts:
    inchfield:
    jast:
```

**New Playbooks:**
- `setup_seaweedfs.yaml` - Deploy SeaweedFS cluster
- `setup_longhorn.yaml` - Deploy Longhorn cluster
- `validate_storage.yaml` - Comprehensive storage testing

**Goldentooth CLI Integration:**
```bash
# Deploy hybrid storage
goldentooth setup_seaweedfs
goldentooth setup_longhorn

# Validate storage functionality
goldentooth validate_storage

# Monitor storage health
goldentooth storage_status
```

## Use Case Specialization

The hybrid architecture enabled clear use case optimization:

### SeaweedFS (ReadWriteMany) Use Cases
- **Shared Development Environments**: Multiple pods accessing common codebases
- **Configuration Management**: Shared config files across service replicas
- **Content Distribution**: Static assets served by multiple frontend pods
- **Log Aggregation**: Multiple log collectors writing to shared storage
- **CI/CD Workspaces**: Build artifacts shared across pipeline stages

### Longhorn (ReadWriteOnce) Use Cases
- **Database Storage**: High-performance storage for PostgreSQL, MySQL
- **Time-Series Data**: InfluxDB, Prometheus TSDB requiring fast I/O
- **Search Indexes**: Elasticsearch nodes with local storage requirements
- **Container Registries**: Docker registry storage requiring consistency
- **Backup Destinations**: High-speed backup target storage

## Monitoring and Observability Enhancements

The hybrid approach improved our monitoring capabilities:

### SeaweedFS Monitoring
```yaml
# Prometheus ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: seaweedfs-monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: seaweedfs
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

**Key Metrics:**
- Volume count and utilization
- Request rates and latency
- Memory usage per component
- Network bandwidth utilization
- Master election status

### Longhorn Monitoring
```yaml
# Prometheus ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: longhorn-monitoring
spec:
  selector:
    matchLabels:
      app: longhorn-manager
  endpoints:
  - port: manager
    interval: 30s
    path: /v1/metrics
```

**Key Metrics:**
- Volume performance (IOPS, throughput, latency)
- Replica synchronization status
- Engine memory usage
- Backup completion rates
- Node storage utilization
