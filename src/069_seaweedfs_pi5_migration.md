# SeaweedFS Pi 5 Migration and CSI Integration

After the successful initial SeaweedFS deployment on the Pi 4B nodes (fenn and karstark), a significant hardware upgrade opportunity arose. Four new Raspberry Pi 5 nodes with 1TB NVMe SSDs had joined the cluster: **Manderly**, **Norcross**, **Oakheart**, and **Payne**. This chapter chronicles the complete migration of the SeaweedFS distributed storage system to these more powerful nodes and the resolution of critical clustering issues that enabled full Kubernetes CSI integration.

## The New Hardware Foundation

### Meet the Storage Powerhouses

The four new Pi 5 nodes represent a massive upgrade in storage capacity and performance:

- **Manderly** (10.4.0.22) - 1TB NVMe SSD via PCIe
- **Norcross** (10.4.0.23) - 1TB NVMe SSD via PCIe
- **Oakheart** (10.4.0.24) - 1TB NVMe SSD via PCIe
- **Payne** (10.4.0.25) - 1TB NVMe SSD via PCIe

**Total Raw Capacity**: 4TB across four nodes (vs. ~1TB across two Pi 4B nodes)

### Performance Characteristics

The Pi 5 + NVMe combination delivers substantial improvements:
- **Storage Interface**: PCIe NVMe vs. USB 3.0 SSD
- **Sequential Read/Write**: ~400MB/s vs. ~100MB/s
- **Random IOPS**: 10x improvement for small file operations
- **CPU Performance**: Cortex-A76 vs. Cortex-A72 cores
- **Memory**: 8GB LPDDR4X vs. 4GB on old nodes

## Migration Strategy

### Cluster Topology Decision

Rather than attempt in-place migration, the decision was made to completely rebuild the SeaweedFS cluster on the new hardware. This approach provided:

1. **Clean Architecture**: No legacy configuration artifacts
2. **Improved Topology**: Optimize for 4-node distributed storage
3. **Zero Downtime**: Keep old cluster running during migration
4. **Rollback Safety**: Ability to revert if issues arose

### Node Role Assignment

The four Pi 5 nodes were configured with hybrid roles to maximize both performance and fault tolerance:

- **Masters**: Manderly, Norcross, Oakheart (3-node Raft consensus)
- **Volume Servers**: All four nodes (maximizing storage capacity)

This design provides proper Raft consensus with an odd number of masters while utilizing all available storage capacity.

## The Critical Discovery: Raft Consensus Requirements

### The Leadership Election Problem

The initial migration attempt using all four nodes as masters immediately revealed a fundamental issue:

```
F0804 21:16:33.246267 master.go:285 Only odd number of masters are supported:
[10.4.0.22:9333 10.4.0.23:9333 10.4.0.24:9333 10.4.0.25:9333]
```

**SeaweedFS requires an odd number of masters for Raft consensus.** This is a fundamental requirement of distributed consensus algorithms to avoid split-brain scenarios where no majority can be established.

### The Mathematics of Consensus

With 4 masters:
- **Split scenarios**: 2-2 splits prevent majority formation
- **Leadership impossible**: No node can achieve >50% votes
- **Cluster paralysis**: "Leader not selected yet" errors continuously

With 3 masters:
- **Majority possible**: 2 out of 3 can form majority
- **Fault tolerance**: 1 node failure still allows operation
- **Clear leadership**: Proper Raft election cycles

## Infrastructure Template Updates

### Fixing Hardcoded Configurations

The migration revealed template issues that needed correction:

#### Dynamic Peer Discovery
```jinja2
# Before (hardcoded)
-peers=fenn:9333,karstark:9333

# After (dynamic)
-peers={% for host in groups['seaweedfs'] %}{{ host }}:9333{% if not loop.last %},{% endif %}{% endfor %}
```

#### Consul Service Template Fix
```json
{
  "peer_addresses": "{% for host in groups['seaweedfs'] %}{{ host }}:9333{% if not loop.last %},{% endif %}{% endfor %}"
}
```

### Removing Problematic Parameters

The `-ip=` parameter in master service templates was causing duplicate peer entries:

```systemd
# Problematic configuration
ExecStart=/usr/local/bin/weed master \
    -port=9333 \
    -mdir=/mnt/seaweedfs-nvme/master \
    -peers=manderly:9333,norcross:9333,oakheart:9333 \
    -ip=10.4.0.22 \                    # <-- This caused duplicates
    -raftHashicorp=true

# Clean configuration
ExecStart=/usr/local/bin/weed master \
    -port=9333 \
    -mdir=/mnt/seaweedfs-nvme/master \
    -peers=manderly:9333,norcross:9333,oakheart:9333 \
    -raftHashicorp=true
```

## Kubernetes CSI Integration Challenge

### The DNS Resolution Problem

With the SeaweedFS cluster running on bare metal and Kubernetes CSI components running in pods, a networking challenge emerged:

**Problem**: Kubernetes pods couldn't resolve SeaweedFS node hostnames because they exist outside the cluster DNS.

**Solution**: Kubernetes Services with explicit Endpoints to bridge the DNS gap.

### Service-Based DNS Resolution

```yaml
# Headless service for each SeaweedFS node
apiVersion: v1
kind: Service
metadata:
  name: manderly
  namespace: default
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: master
    port: 9333
  - name: volume
    port: 8080
---
# Explicit endpoint mapping
apiVersion: v1
kind: Endpoints
metadata:
  name: manderly
  namespace: default
subsets:
- addresses:
  - ip: 10.4.0.22
  ports:
  - name: master
    port: 9333
  - name: volume
    port: 8080
```

This approach allows the SeaweedFS filer (running in Kubernetes) to connect to the bare metal cluster using service names like `manderly:9333`.

## Migration Execution

### Phase 1: Infrastructure Preparation

```bash
# Update inventory to reflect new nodes
goldentooth edit_vault
# Configure new SeaweedFS group with Pi 5 nodes

# Clean deployment of storage infrastructure
goldentooth cleanup_old_storage
goldentooth setup_seaweedfs
```

### Phase 2: Cluster Formation with Proper Topology

```bash
# Deploy 3-master configuration
goldentooth command_root manderly,norcross,oakheart "systemctl start seaweedfs-master"

# Verify leadership election
curl http://10.4.0.22:9333/dir/status

# Start volume servers on all nodes
goldentooth command_root manderly,norcross,oakheart,payne "systemctl start seaweedfs-volume"
```

### Phase 3: Kubernetes Integration

```bash
# Deploy DNS bridge services
kubectl apply -f seaweedfs-services-endpoints.yaml

# Deploy and verify filer
kubectl get pods -l app=seaweedfs-filer
kubectl logs seaweedfs-filer-xxx | grep "Start Seaweed Filer"
```

## Verification and Testing

### Cluster Health Verification

```bash
# Leadership confirmation
curl http://10.4.0.22:9333/cluster/status
# Returns proper topology with elected leader

# Service status across all nodes
goldentooth command manderly,norcross,oakheart,payne "systemctl status seaweedfs-master seaweedfs-volume"
```

### CSI Integration Testing

```yaml
# Test PVC creation
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: seaweedfs-test-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
  storageClassName: seaweedfs-storage
```

**Result**: Successful dynamic volume provisioning with NFS-style mounting via `seaweedfs-filer:8888:/buckets/pvc-xxx`.

### End-to-End Functionality

```bash
# Pod with mounted SeaweedFS volume
kubectl exec test-pod -- df -h /data
# Filesystem: seaweedfs-filer:8888:/buckets/pvc-xxx Size: 512M

# File I/O verification
kubectl exec test-pod -- touch /data/test-file
kubectl exec test-pod -- ls -la /data/
# Files persist across pod restarts via distributed storage
```

## Final Architecture

### Cluster Topology

- **Masters**: 3 nodes (Manderly, Norcross, Oakheart) with Raft consensus
- **Volume Servers**: 4 nodes (all Pi 5s) with 1TB NVMe each
- **Total Capacity**: ~3.6TB usable distributed storage
- **Fault Tolerance**: Can survive 1 master failure + multiple volume server failures
- **Performance**: NVMe speeds with distributed redundancy

### Integration Status

- ✅ **Kubernetes CSI**: Dynamic volume provisioning working
- ✅ **DNS Resolution**: Service-based hostname resolution
- ✅ **Leadership Election**: Stable Raft consensus
- ✅ **Filer Services**: HTTP/gRPC endpoints operational
- ✅ **Volume Mounting**: NFS-style filesystem access
- ✅ **High Availability**: Multi-node fault tolerance

### Monitoring Integration

SeaweedFS metrics integrate with the existing Goldentooth observability stack:

- **Prometheus**: Master and volume server metrics collection
- **Grafana**: Storage capacity and performance dashboards
- **Consul**: Service discovery and health monitoring
- **Step-CA**: TLS certificate management for secure communications

## Performance Impact

### Storage Capacity Comparison

| Metric            | Old Cluster (Pi 4B) | New Cluster (Pi 5) | Improvement |
| ----------------- | ------------------- | ------------------ | ----------- |
| Total Capacity    | ~1TB                | ~3.6TB             | 3.6x        |
| Node Count        | 2                   | 4                  | 2x          |
| Per-Node Storage  | 500GB               | 1TB                | 2x          |
| Storage Interface | USB 3.0 SSD         | PCIe NVMe          | PCIe speed  |
| Fault Tolerance   | Single failure      | Multi-failure      | Higher      |

### Architectural Benefits

- **Proper Consensus**: 3-master Raft eliminates split-brain scenarios
- **Expanded Capacity**: 3.6TB enables larger workloads and datasets
- **Performance Scaling**: NVMe storage handles high-IOPS workloads
- **Kubernetes Native**: CSI integration enables GitOps storage workflows
- **Future Ready**: Foundation for S3 gateway and advanced SeaweedFS features
