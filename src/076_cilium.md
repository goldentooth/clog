# Cilium CNI: eBPF-Based Networking

After migrating to Talos Linux (see [070_talos](./070_talos.md)), the cluster was running with Flannel as the Container Network Interface (CNI) and kube-proxy for service load balancing. While this worked, it wasn't taking advantage of modern eBPF capabilities or the full potential of Talos's networking features.

Enter Cilium: an eBPF-based CNI that replaces both Flannel and kube-proxy with more efficient, observable, and feature-rich networking.

## Why Cilium?

Cilium offers several advantages over traditional CNIs:

- **eBPF-based data plane**: More efficient than iptables-based solutions
- **KubeProxy replacement**: Talos integrates with Cilium's kube-proxy replacement via KubePrism
- **Hubble observability**: Deep network flow visibility without external tooling
- **L7-aware policies**: HTTP/gRPC-level network policies and load balancing
- **Bandwidth management**: Built-in traffic shaping with BBR congestion control
- **Host firewall**: eBPF-based firewall on each node
- **Encryption ready**: Optional WireGuard transparent encryption

For a learning cluster, Cilium provides a wealth of features to explore and experiment with.

## Talos-Specific Configuration

Talos requires specific CNI configuration that differs from standard Kubernetes:

### 1. Disable Built-in CNI

In `talconfig.yaml`, the CNI must be explicitly disabled:

```yaml
patches:
  - |-
    cluster:
      network:
        cni:
          name: none
      proxy:
        disabled: true
```

This tells Talos not to deploy its own CNI or kube-proxy, leaving that responsibility to Cilium.

### 2. Security Context Adjustments

Talos doesn't allow pods to load kernel modules, so the `SYS_MODULE` capability must be removed from Cilium's security context. The eBPF programs don't require this capability anyway.

### 3. Cgroup Configuration

Talos uses cgroupv2 and mounts it at `/sys/fs/cgroup`. Cilium needs to be told to use these existing mounts rather than trying to mount its own:

```yaml
cgroup:
  autoMount:
    enabled: false
  hostRoot: /sys/fs/cgroup
```

### 4. KubePrism Integration

Talos replaces kube-proxy with KubePrism, which runs on localhost:7445. Cilium's kube-proxy replacement needs to know about this:

```yaml
k8sServiceHost: localhost
k8sServicePort: 7445
kubeProxyReplacement: true
```

## GitOps Deployment with Flux

Rather than using the Cilium CLI installer, I deployed Cilium via Flux HelmRelease to maintain the GitOps approach. This ensures the CNI configuration is version-controlled and reproducible.

The structure mirrors other infrastructure components:

```
gitops/infrastructure/cilium/
├── kustomization.yaml
├── namespace.yaml
├── release.yaml
└── repository.yaml
```

The HelmRelease (`release.yaml`) contains all the Cilium configuration, including:

- Talos-specific settings (covered above)
- IPAM mode set to `kubernetes` (required for Talos)
- Hubble observability enabled
- Bandwidth manager with BBR
- L7 proxy for HTTP/gRPC features
- Host firewall enabled
- Network tunnel mode (VXLAN)

## Migration Process

The migration from Flannel to Cilium was surprisingly straightforward:

1. **Created GitOps manifests**: Added Cilium HelmRelease to the infrastructure kustomization
2. **Pushed to Git**: Committed the changes to trigger Flux reconciliation
3. **Regenerated Talos configs**: Ran `talhelper genconfig` to generate new machine configurations with CNI disabled
4. **Applied configuration**: Used `talhelper gencommand apply | bash` to apply to all nodes
5. **Waited for Cilium**: Nodes rebooted and waited (hung on phase 18/19 as expected) until Cilium pods started
6. **Removed old CNI**: Deleted the Flannel and kube-proxy DaemonSets:
   ```bash
   kubectl delete daemonset -n kube-system kube-flannel
   kubectl delete daemonset -n kube-system kube-proxy
   ```

The cluster had a brief period where both CNIs were running simultaneously, which caused some endpoint conflicts. Once Flannel was removed, Cilium took over completely and networking stabilized.

## Verification

After the migration, verification confirmed everything was working:

```bash
$ kubectl exec -n kube-system ds/cilium -- cilium status --brief
OK
```

All 17 nodes came back as Ready, with Cilium agent and Envoy pods running on each:

- 17 Cilium agents (main CNI component)
- 17 Cilium Envoy proxies (L7 proxy)
- 1 Cilium Operator (cluster-wide coordination)
- 1 Hubble Relay (flow aggregation)
- 1 Hubble UI (web interface for flow visualization)

Testing connectivity with a simple pod confirmed networking was functional:

```bash
$ kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -O- -T 5 https://google.com
Connecting to google.com (192.178.154.139:443)
writing to stdout
<!doctype html><html itemscope="" itemtype="http://schema.org/WebPage" lang="en">...
```

DNS resolution and external connectivity both worked perfectly.

## Features Enabled

The deployment enables several advanced features out of the box:

### Hubble Observability

Hubble provides deep visibility into network flows without requiring external tools or sidecar proxies. It captures:

- DNS queries and responses
- TCP connection establishment and teardown
- HTTP request/response metadata
- Packet drops with detailed reasons
- ICMP messages

Initially configured as ClusterIP, the Hubble UI was later exposed via LoadBalancer (see "Exposing Hubble UI" section below).

### Bandwidth Manager

Cilium's bandwidth manager uses eBPF to implement efficient traffic shaping with BBR congestion control. This provides better network utilization than traditional tc-based solutions.

### Host Firewall

The eBPF-based host firewall runs on each node, providing network filtering without the overhead of iptables.

### L7 Proxy

Cilium Envoy pods provide L7 load balancing and policy enforcement for HTTP and gRPC traffic. This enables features like HTTP header-based routing and gRPC method-level policies.

## Additional Features Available

### BGP Control Plane

For advanced routing scenarios, Cilium's BGP control plane can peer with network equipment:

```yaml
bgpControlPlane:
  enabled: true
```

This would allow the cluster to advertise Pod CIDRs directly to the network infrastructure.

## Challenges Encountered

### Dual CNI Period

Having both Flannel and Cilium running simultaneously caused endpoint conflicts. The Kubernetes endpoint controller struggled to update endpoints, resulting in timeout errors:

```
Failed to update endpoint kube-system/hubble-peer: etcdserver: request timed out
```

Once Flannel was removed, these issues resolved immediately.

### Stale CNI Configuration

Some pods that were created during the Flannel period tried to use the old CNI configuration. The Hubble UI pod, for example, failed with:

```
plugin type="flannel" failed (add): loadFlannelSubnetEnv failed:
open /run/flannel/subnet.env: no such file or directory
```

Deleting and recreating these pods resolved the issue.

### Flannel Interface Remnants

The `flannel.1` interface remains on nodes even after removing Flannel. Cilium notices it and includes it in the host firewall configuration. This is harmless but obviously I'm gonna clean that up.

### PodSecurity Policy Blocks

The `cilium connectivity test` command initially failed because Talos enforces a "baseline" PodSecurity policy by default. The test pods require the `NET_RAW` capability, which is blocked by this policy:

```
Error creating: pods "client-64d966fcbd-sd4lw" is forbidden:
violates PodSecurity "baseline:latest": non-default capabilities
(container "client" must not include "NET_RAW" in securityContext.capabilities.add)
```

The solution is to use the `--namespace-labels` flag to set the test namespace to "privileged" mode:

```bash
cilium connectivity test \
  --namespace-labels pod-security.kubernetes.io/enforce=privileged,pod-security.kubernetes.io/audit=privileged,pod-security.kubernetes.io/warn=privileged
```

## Exposing Hubble UI

Rather than requiring port-forwarding to access Hubble UI, the service was configured as a LoadBalancer with External-DNS annotation:

```yaml
hubble:
  ui:
    enabled: true
    service:
      type: LoadBalancer
      annotations:
        external-dns.alpha.kubernetes.io/hostname: hubble.goldentooth.net
```

After applying this change via Flux:

1. MetalLB assigned an IP from the pool (10.4.11.2)
2. External-DNS created a DNS record in Route53
3. Hubble UI became accessible at `http://hubble.goldentooth.net`

This follows the same pattern as other services in the cluster, maintaining consistency and eliminating manual port-forwarding steps.

## Enabling WireGuard Encryption

After confirming Cilium was working correctly, WireGuard encryption was enabled to provide transparent pod-to-pod encryption across nodes:

```yaml
encryption:
  enabled: true
  type: wireguard
  nodeEncryption: false
```

The `nodeEncryption: false` setting means only pod traffic is encrypted, not host-level communication. This is typically the desired behavior for most clusters.

### Rollout Process

Enabling encryption triggered a rolling update of all Cilium agents:

```bash
$ kubectl rollout status daemonset/cilium -n kube-system
Waiting for daemon set "cilium" rollout to finish: 11 out of 17 new pods have been updated...
Waiting for daemon set "cilium" rollout to finish: 12 out of 17 new pods have been updated...
...
daemon set "cilium" successfully rolled out
```

The rollout took several minutes as each node's Cilium agent was updated with the new encryption configuration.

### Verification

After the rollout completed, verification confirmed WireGuard was active:

```bash
$ kubectl exec -n kube-system ds/cilium -- cilium status | grep -i encrypt
Encryption: Wireguard [NodeEncryption: Disabled, cilium_wg0
  (Pubkey: DKyUQtNylJpQv6xf9cnbuvcCgCeahumOcOE5cfxt/kk=, Port: 51871, Peers: 16)]
```

This output shows:
- WireGuard is active on interface `cilium_wg0`
- Each node has 16 peers (the other nodes in the cluster)
- The WireGuard tunnel is listening on port 51871
- A unique public key was generated for this node

All pod-to-pod traffic across nodes is now transparently encrypted with WireGuard's ChaCha20-Poly1305 cipher, with zero application changes required.
