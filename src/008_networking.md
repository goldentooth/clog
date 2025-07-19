# Networking

Kubernetes uses three different networks:

- Infrastructure: The physical or virtual backbone connecting the machines hosting the nodes. The infrastructure network enables connectivity between the nodes; this is essential for the Kubernetes control plane components (like the kube-apiserver, etcd, scheduler, and controller-manager) and the worker nodes to communicate with each other. Although pods communicate with each other via the pod network (overlay network), the underlying infrastructure network supports this by facilitating the physical or virtual network paths between nodes.
- Service: This is a purely virtual and internal network. It allows services to communicate with each other and with Pods seamlessly. This network layer abstracts the actual network details from the services, providing a consistent and simplified interface for inter-service communication. When a Service is created, it is automatically assigned a unique IP address from the service network's address space. This IP address is stable for the lifetime of the Service, even if the Pods that make up the Service change. This stable IP address makes it easier to configure DNS or other service discovery mechanisms.
- Pod: This is a crucial component that allows for seamless communication between pods across the cluster, regardless of which node they are running on. This networking model is designed to ensure that each pod gets its own unique IP address, making it appear as though each pod is on a flat network where every pod can communicate with every other pod directly without NAT.

My infrastructure network is already up and running at `10.4.0.0/20`. I'll configure my service network at `172.16.0.0/20` and my pod network at `192.168.0.0/16`.

## Network Architecture Implementation

### CIDR Block Allocations

The goldentooth cluster uses a carefully planned network segmentation strategy:

- **Infrastructure Network**: `10.4.0.0/20` - Physical network backbone
- **Service Network**: `172.16.0.0/20` - Kubernetes virtual services  
- **Pod Network**: `192.168.0.0/16` - Container-to-container communication
- **MetalLB Range**: `10.4.11.0/24` - Load balancer service IPs

### Physical Network Topology

The cluster consists of:

**Control Plane Nodes (High Availability)**:
- bettley (10.4.0.11), cargyll (10.4.0.12), dalt (10.4.0.13)

**Load Balancer and Services**:
- allyrion (10.4.0.10) - HAProxy load balancer, NFS server

**Worker Nodes**:
- 8 Raspberry Pi ARM64 workers: erenford, fenn, gardener, harlton, inchfield, jast, karstark, lipps
- 1 x86 GPU worker: velaryon (10.4.0.30)

### CNI Implementation: Calico

The cluster uses [Calico](https://projectcalico.docs.tigera.io/) as the Container Network Interface (CNI) plugin. Calico is configured during the kubeadm initialization:

```bash
kubeadm init \
  --control-plane-endpoint="10.4.0.10:6443" \
  --service-cidr="172.16.0.0/20" \
  --pod-network-cidr="192.168.0.0/16" \
  --kubernetes-version="stable-1.32"
```

Calico provides:
- Layer 3 networking with BGP routing
- Network policies for microsegmentation
- Cross-node pod communication without overlay networks
- Integration with the existing BGP infrastructure

### Load Balancer Architecture

**HAProxy Configuration**:
The cluster uses HAProxy running on allyrion (10.4.0.10) to provide high availability for the Kubernetes API server:

- **Frontend**: Listens on port 6443
- **Backend**: Round-robin load balancing across all three control plane nodes
- **Health Checks**: TCP-based health checks with fall=3, rise=2 configuration
- **Monitoring**: Prometheus metrics endpoint on port 8405

This ensures the cluster remains accessible even if individual control plane nodes fail.

### BGP Integration with MetalLB

The cluster implements BGP-based load balancing using MetalLB:

**Router Configuration** (OPNsense with FRR):
- **Router AS Number**: 64500
- **Cluster AS Number**: 64501  
- **BGP Peer**: Router at 10.4.0.1

**MetalLB Configuration**:
```yaml
spec:
  myASN: 64501
  peerASN: 64500
  peerAddress: 10.4.0.1
  addressPool: '10.4.11.0 - 10.4.15.254'
```

This allows Kubernetes LoadBalancer services to receive real IP addresses that are automatically routed through the network infrastructure.

### Static Route Management

The networking Ansible role automatically:
1. Detects the primary network interface using `ip route show 10.4.0.0/20`
2. Adds static routes for the MetalLB range: `ip route add 10.4.11.0/24 dev <interface>`
3. Persists routes in `/etc/network/interfaces.d/<interface>.cfg` for boot persistence

### Service Discovery and DNS

The cluster implements comprehensive service discovery:

- **Cluster Domain**: `goldentooth.net`
- **Node Domain**: `nodes.goldentooth.net`
- **Services Domain**: `services.goldentooth.net`
- **External DNS**: Automated DNS record management via external-dns operator

### Network Security

**Certificate-Based Security**:
- **Step-CA**: Provides automated certificate management for all services
- **TLS Everywhere**: All inter-service communication is encrypted
- **SSH Certificates**: Automated SSH certificate provisioning

**Service Mesh Integration**:
- **Consul**: Provides service discovery and health checking across both Kubernetes and Nomad
- **Network Policies**: Configured but not strictly enforced by default

### Multi-Orchestrator Networking

The cluster supports both Kubernetes and HashiCorp Nomad workloads on the same physical network:

- **Kubernetes**: Calico CNI with BGP routing
- **Nomad**: Bridge networking with Consul Connect service mesh
- **Vault**: Network-based authentication and secrets distribution

### Monitoring Network Integration

**Observability Stack**:
- **Prometheus**: Scrapes metrics across all network endpoints
- **Grafana**: Centralized dashboards accessible via MetalLB LoadBalancer
- **Loki**: Log aggregation with Vector log shipping across nodes
- **Node Exporter**: Per-node metrics collection

With this network architecture decided and implemented, we can move forward to the next phase of cluster construction.
