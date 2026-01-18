# Cluster Hardware

I went with a [PicoCluster 10H](https://www.picocluster.com/products/copy-of-pico-10-raspberry-pi4-8gb). I'm well aware that I could've cobbled something together and spent much less money; I have indeed done the thing with a bunch of Raspberry Pis screwed to a board and plugged into an Anker USB charger and a TP-Link switch.

I didn't want to do that again, though. For one, I've experienced problems with USB chargers seeming to lose power over time, and some small switches getting flaky when powered from USB. I liked the power supply of the PicoCluster and its cooling configuration. I liked that it did pretty much exactly what I wanted, and if I had problems I could yell at someone else about it rather than getting derailed by hardware rabbit holes.

I also purchased ten large heatsinks with fans, specifically [these](https://www.amazon.com/gp/product/B091L1XKL6?ie=UTF8&psc=1). There were others I liked a bit more, and these interfered with the standoffs that were used to build each stack of five Raspberry Pis, but these seemed as though they would likely be the most reliable in the long run.

I purchased SanDisk 128GB Extreme microSDXC cards for local storage. I've been using SanDisk cards for years with no significant issues or complaints.

The individual nodes are Raspberry Pi 4B/8GB. As of the time I'm writing this, Raspberry Pi 5s are out, and they offer very substantial benefits over the 4B. That said, they also have higher energy consumption, lower availability, and so forth. I'm opting for a lower likelihood of surprises because, again, I just don't want to spend much time dealing with hardware and I don't expect performance to hinder me.

## Technical Specifications

### Complete Node Inventory

The cluster consists of 13 nodes with specific roles and configurations:

**Raspberry Pi Nodes (12 total)**:
- **allyrion** (10.4.0.10) - Pi4B - NFS server, HAProxy load balancer, Docker host
- **bettley** (10.4.0.11) - Pi4B - Kubernetes control plane, Consul server, Vault server
- **cargyll** (10.4.0.12) - Pi4B - Kubernetes control plane, Consul server, Vault server
- **dalt** (10.4.0.13) - Pi4B - Kubernetes control plane, Consul server, Vault server
- **erenford** (10.4.0.14) - Pi4B - Kubernetes worker, Ray head node, ZFS storage
- **fenn** (10.4.0.15) - Pi4B - Kubernetes worker, Ceph storage node
- **gardener** (10.4.0.16) - Pi4B - Kubernetes worker, Grafana host, ZFS storage
- **harlton** (10.4.0.17) - Pi4B - Kubernetes worker
- **inchfield** (10.4.0.18) - Pi5 - Kubernetes worker, Loki log aggregation
- **jast** (10.4.0.19) - Pi5 - Kubernetes worker, Step-CA certificate authority
- **karstark** (10.4.0.20) - Pi5 - Kubernetes worker, Ceph storage node
- **lipps** (10.4.0.21) - Pi5 - Kubernetes worker, Ceph storage node

**x86 GPU Node**:
- **velaryon** (10.4.0.30) - AMD Ryzen 9 3900X, 32GB RAM, NVIDIA RTX 2070 Super

### Hardware Architecture

**Raspberry Pi 4B Specifications**:
- **CPU**: ARM Cortex-A72 quad-core @ 2.0GHz (overclocked from 1.5GHz)
- **RAM**: 8GB LPDDR4
- **Storage**: SanDisk 128GB Extreme microSDXC (UHS-I Class 10)
- **Network**: Gigabit Ethernet (onboard)
- **GPIO**: Used for fan control (pin 14) and hardware monitoring

**Performance Optimizations**:
```
arm_freq=2000
over_voltage=6
```

These overclocking settings provide approximately 33% performance increase while maintaining thermal stability with active cooling.

**Raspberry Pi 5 Specifications**:
- **CPU**: ARM Cortex-A76 quad-core @ 2.4GHz
- **RAM**: 8GB LPDDR4X
- **Storage**: SanDisk 256GB Extreme microSDXC (UHS-I Class 10), 1TB NVMe SSD
- **Network**: Gigabit Ethernet (onboard)
- **GPIO**: Used for fan control (pin 14) and hardware monitoring

### Network Infrastructure

**Network Segmentation**:
- **Infrastructure CIDR**: `10.4.0.0/20` - Physical network backbone
- **Service CIDR**: `172.16.0.0/20` - Kubernetes virtual services
- **Pod CIDR**: `192.168.0.0/16` - Container networking
- **MetalLB Range**: `10.4.11.0/24` - Load balancer IP allocation

**MAC Address Registry**:
Each node has documented MAC addresses for network boot and management:
- Raspberry Pi nodes: `d8:3a:dd:*` and `dc:a6:32:*` prefixes
- x86 node: `2c:f0:5d:0f:ff:39` (velaryon)

### Storage Architecture

**Distributed Storage Strategy**:

**NFS Shared Storage**:
- **Server**: allyrion exports `/mnt/usb1`
- **Clients**: All 13 nodes mount at `/mnt/nfs`
- **Use Cases**: Configuration files, shared datasets, cluster coordination

**ZFS Storage Pool**:
- **Nodes**: allyrion, erenford, gardener
- **Pool**: `rpool` with `rpool/data` dataset
- **Features**: Snapshots, replication, compression
- **Optimization**: 128MB ARC limit for Raspberry Pi RAM constraints

**Ceph Distributed Storage**:
- **Nodes**: fenn, karstark, lipps
- **Purpose**: Highly available distributed block and object storage
- **Integration**: Kubernetes persistent volumes

### Thermal Management

**Cooling Configuration**:
- **Heatsinks**: Large aluminum heatsinks with 40mm fans per node
- **Fan Control**: GPIO-based temperature control at 60°C threshold
- **Airflow**: PicoCluster chassis provides directed airflow path
- **Monitoring**: Temperature sensors exposed via Prometheus metrics

**Thermal Performance**:
- **Idle**: ~45-50°C ambient
- **Load**: ~60-65°C under sustained workload
- **Throttling**: No thermal throttling observed during normal operations

### Power Architecture

**Power Supply**:
- **Input**: Single AC connection to PicoCluster power distribution
- **Per Node**: 5V/3A regulated power (avoiding USB charger degradation)
- **Efficiency**: ~90% efficiency at typical load
- **Redundancy**: Single point of failure by design (acceptable for lab environment)

**Power Consumption**:
- **Raspberry Pi**: ~8W idle, ~15W peak per node
- **Total Pi Load**: ~96W idle, ~180W peak (12 nodes)
- **x86 Node**: ~150W idle, ~300W peak
- **Cluster Total**: ~250W idle, ~480W peak

### Hardware Monitoring

**Metrics Collection**:
- **Node Exporter**: Hardware sensors, thermal data, power metrics
- **Prometheus**: Centralized metrics aggregation
- **Grafana**: Real-time dashboards with thermal and performance alerts

**Monitored Parameters**:
- CPU temperature and frequency
- Memory usage and availability
- Storage I/O and capacity
- Network interface statistics
- Fan speed and cooling device status

### Reliability Considerations

**Hardware Resilience**:
- **No RAID**: Individual node failure acceptable (distributed applications)
- **Network Redundancy**: Single switch (acceptable for lab)
- **Power Redundancy**: Single PSU (lab environment limitation)
- **Cooling Redundancy**: Individual fan failure affects single node only

**Failure Recovery**:
- **Kubernetes**: Automatic pod rescheduling on node failure
- **Consul/Vault**: Multi-node quorum survives single node loss
- **Storage**: ZFS replication and Ceph redundancy provide data protection

### Future Expansion

**Planned Upgrades**:
- **SSD Storage**: USB 3.0 SSD expansion for high-IOPS workloads
- **Network Upgrades**: Potential 10GbE expansion via USB adapters
- **Additional GPU**: PCIe expansion for ML workloads

## Frequently Asked Questions

### So, how do you like the PicoCluster so far?

I have no complaints. Putting it together was straightforward; the documentation was great, everything was labeled correctly, etc. Cooling seems very adequate and performance and appearance are perfect.

The integrated power supply has been particularly reliable compared to previous experiences with USB charger-based setups. The structured cabling and chassis design make maintenance and monitoring much easier than ad-hoc Raspberry Pi clusters.

### Have you considered adding SSDs for mass storage?

Yes, and I have some cables and spare SSDs for doing so. I'm not sure if I actually _will_. We'll see.

The current storage architecture with ZFS pools on USB-attached SSDs and distributed Ceph storage has proven adequate for most workloads. The microSD cards handle the OS and container storage well, while shared storage needs are met through the NFS and distributed storage layers.
