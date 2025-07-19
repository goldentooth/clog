# Fixing MetalLB

As mentioned [here](./027_welcome_back.md), I purchased a new router to replace a power-hungry Dell server running OPNsense, and that cost me BGP support. This kills my MetalLB configuration, so I need to switch it to use Layer 2.

This transition represents a fundamental change in how MetalLB operates and requires understanding the trade-offs between BGP and Layer 2 modes.

## BGP vs Layer 2 Architecture Comparison

### BGP Mode (Previous Configuration)
- **Dynamic routing**: BGP speakers advertise LoadBalancer IPs to upstream routers
- **True load balancing**: Multiple nodes can announce the same service IP with ECMP
- **Scalability**: Router handles load distribution and failover automatically
- **Network integration**: Works with enterprise routing infrastructure
- **Requirements**: Router must support BGP (FRR, Quagga, hardware routers)

### Layer 2 Mode (New Configuration)
- **ARP announcements**: Nodes respond to ARP requests for LoadBalancer IPs
- **Active/passive failover**: Only one node answers ARP for each service IP
- **Simpler setup**: No routing protocol configuration required
- **Limited scalability**: All traffic for a service goes through single node
- **Requirements**: Nodes must be on same Layer 2 network segment

## Hardware Infrastructure Change

The transition was necessitated by hardware changes:

**Previous Setup**:
- **Dell server**: Power-hungry (likely PowerEdge) running OPNsense
- **BGP support**: FRR (Free Range Routing) plugin provided full BGP implementation
- **Power consumption**: High power draw from server-class hardware
- **Complexity**: Full routing stack with BGP, OSPF, and other protocols

**New Setup**:
- **Consumer router**: Lower power consumption
- **No BGP support**: Consumer-grade firmware lacks routing protocol support
- **Simplified networking**: Standard static routing and NAT
- **Cost efficiency**: Reduced power costs and hardware complexity

## Migration Process

The migration involved several coordinated steps to minimize service disruption:

### Step 1: Remove BGP Configuration

That shouldn't be too bad.

I think it's just a matter of deleting the BGP advertisement:

```bash
$ sudo kubectl -n metallb delete BGPAdvertisement primary
bgpadvertisement.metallb.io "primary" deleted
```

This command removes the BGP advertisement configuration, which:
- **Stops route announcements**: MetalLB speakers stop advertising LoadBalancer IPs via BGP
- **Maintains IP allocation**: Existing LoadBalancer services keep their assigned IPs
- **Preserves connectivity**: Services remain accessible until Layer 2 mode is configured

### Step 2: Configure Layer 2 Advertisement

and creating an L2 advertisement:

```bash
$ cat tmp.yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: primary
  namespace: metallb

$ sudo kubectl apply -f tmp.yaml
l2advertisement.metallb.io/primary created
```

**L2Advertisement Configuration Details**:
```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: primary
  namespace: metallb
spec:
  ipAddressPools:
  - primary
  nodeSelectors:
  - matchLabels:
      kubernetes.io/hostname: "*"
  interfaces:
  - eth0
```

**Key behaviors in Layer 2 mode**:
- **ARP responder**: Nodes respond to ARP requests for LoadBalancer IPs
- **Leader election**: One node per service IP elected as ARP responder
- **Gratuitous ARP**: Leader sends gratuitous ARP to announce IP ownership
- **Failover**: New leader elected if current leader becomes unavailable

### Step 3: Router Static Route Configuration

After adding the static route to my router, I can see the friendly `go-httpbin` response when I navigate to https://10.4.11.1/

**Static Route Configuration**:
```bash
# Router configuration (varies by model)
# Destination: 10.4.11.0/24 (MetalLB IP pool)
# Gateway: 10.4.0.X (any cluster node IP)
# Interface: LAN interface connected to cluster network
```

**Why static routes are necessary**:
- **IP pool isolation**: MetalLB pool (`10.4.11.0/24`) is separate from cluster network (`10.4.0.0/20`)
- **Router awareness**: Router needs to know how to reach LoadBalancer IPs
- **Return path**: Ensures bidirectional connectivity for external clients

## Network Topology Changes

### Layer 2 Network Requirements

**Physical topology**:
```
[Internet] → [Router] → [Switch] → [Cluster Nodes]
                ↓
         Static Route:
         10.4.11.0/24 → cluster
```

**ARP behavior**:
1. **Client request**: External client sends packet to LoadBalancer IP
2. **Router forwarding**: Router forwards based on static route to cluster network
3. **ARP resolution**: Router/switch broadcasts ARP request for LoadBalancer IP
4. **Node response**: MetalLB leader node responds with its MAC address
5. **Traffic delivery**: Subsequent packets sent directly to leader node

### Failover Mechanism

**Leader election process**:
```bash
# Check current leader for a service
kubectl -n metallb logs -l component=speaker | grep "announcing"

# Example output:
# {"level":"info","ts":"2024-01-15T10:30:00Z","msg":"announcing","ip":"10.4.11.1","node":"bettley"}
```

**Failover sequence**:
1. **Leader failure**: Current announcing node becomes unavailable
2. **Detection**: MetalLB speakers detect leader absence (typically 10-30 seconds)
3. **Election**: Remaining speakers elect new leader using deterministic algorithm
4. **Gratuitous ARP**: New leader sends gratuitous ARP to update network caches
5. **Service restoration**: Traffic resumes through new leader node

## DNS Infrastructure Migration

I also lost some control over DNS, e.g. the router's DNS server will override all lookups for hellholt.net rather than forwarding requests to my DNS servers.

So I created a new domain, goldentooth.net, to handle this cluster. A couple of tweaks to ExternalDNS and some service definitions and I can verify that ExternalDNS is setting the DNS records correctly, although I don't seem to be able to resolve names just yet.

### Domain Migration Impact

**Previous Domain**: `hellholt.net`
- **Router control**: New router overrides DNS resolution
- **Local DNS interference**: Router's DNS server intercepts queries
- **Limited delegation**: Consumer router lacks sophisticated DNS forwarding

**New Domain**: `goldentooth.net`
- **External control**: Managed entirely in AWS Route53
- **Clean delegation**: No local DNS interference
- **ExternalDNS compatibility**: Full automation support

### ExternalDNS Configuration Updates

**Domain filter change**:
```yaml
# Previous configuration
args:
- --domain-filter=hellholt.net

# New configuration  
args:
- --domain-filter=goldentooth.net
```

**Service annotation updates**:
```yaml
# httpbin service example
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: httpbin.goldentooth.net
    # Previously: httpbin.hellholt.net
```

**DNS record verification**:
```bash
# Check Route53 records
aws route53 list-resource-record-sets --hosted-zone-id Z0736727S7ZH91VKK44A

# Verify DNS propagation
dig A httpbin.goldentooth.net
dig TXT httpbin.goldentooth.net  # Ownership records
```

## Performance and Operational Considerations

### Layer 2 Mode Limitations

**Single point of failure**:
- Only one node handles traffic for each LoadBalancer IP
- Node failure causes service interruption until failover completes
- No load distribution across multiple nodes

**Network broadcast traffic**:
- ARP announcements increase broadcast traffic
- Gratuitous ARP during failover events
- Potential impact on large Layer 2 domains

**Scalability constraints**:
- All service traffic passes through single node
- Node bandwidth becomes bottleneck for high-traffic services
- Limited horizontal scaling compared to BGP mode

### Monitoring and Troubleshooting

**MetalLB speaker logs**:
```bash
# Monitor speaker activities
kubectl -n metallb logs -l component=speaker --tail=50

# Check for leader election events
kubectl -n metallb logs -l component=speaker | grep -E "(leader|announcing|failover)"

# Verify ARP announcements
kubectl -n metallb logs -l component=speaker | grep "gratuitous ARP"
```

**Network connectivity testing**:
```bash
# Test ARP resolution for LoadBalancer IPs
arping -c 3 10.4.11.1

# Check MAC address consistency
arp -a | grep "10.4.11"

# Verify static routes on router
ip route show | grep "10.4.11.0/24"
```

### Future TLS Strategy

I think I still need to get TLS working too, but I've soured on the idea of maintaining a cert per domain name and per service. I think I'll just have a wildcard over goldentooth.net and share that out. Too much aggravation otherwise. That's a problem for another time, though.

**Wildcard certificate benefits**:
- **Simplified management**: Single certificate for all subdomains
- **Reduced complexity**: No per-service certificate automation
- **Cost efficiency**: One certificate instead of multiple Let's Encrypt certs
- **Faster deployment**: No certificate provisioning delays for new services

**Implementation considerations**:
```yaml
# Wildcard certificate configuration
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: goldentooth-wildcard
  namespace: default
spec:
  secretName: goldentooth-wildcard-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - "*.goldentooth.net"
  - "goldentooth.net"
```

## Configuration Persistence

The Layer 2 configuration is maintained in the gitops repository structure:

**MetalLB Helm chart updates**:
```yaml
# values.yaml changes
spec:
  # BGP configuration removed
  # bgpPeers: []
  # bgpAdvertisements: []
  
  # Layer 2 configuration added
  l2Advertisements:
  - name: primary
    ipAddressPools:
    - primary
```

This transition demonstrates the flexibility of MetalLB to adapt to different network environments while maintaining service availability. While Layer 2 mode has limitations compared to BGP, it provides a viable solution for simpler network infrastructures and reduces operational complexity in exchange for some scalability constraints.
