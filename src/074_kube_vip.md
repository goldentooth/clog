# Kube-VIP for Control Plane High Availability

With the Talos cluster up and running, I wanted to eliminate a potential single point of failure: the control plane endpoint. While I had three control plane nodes (allyrion, bettley, and cargyll), my DNS record `cp.k8s.goldentooth.net` was configured as a simple round-robin across all three IPs. This works, but it's not ideal—if one node goes down, clients would still try to connect to it until the DNS record updated.

The solution? A Virtual IP (VIP) that floats across the control plane nodes, managed by kube-vip.

## Why Kube-VIP?

Kube-vip is beautifully simple: it uses etcd leader election to decide which control plane node "owns" the VIP at any given time. The winning node responds to requests on that IP. If that node fails:
- **Graceful shutdown**: The VIP migrates almost instantly
- **Unexpected failure**: Failover takes about a minute (by design, to avoid split-brain scenarios)

The best part? No external load balancer hardware required. The VIP is managed entirely within the cluster itself.

## The Chicken-and-Egg Problem

Here's the catch: kube-vip relies on etcd for leader election, so the VIP won't come alive until the cluster is already bootstrapped. This means you can't use the VIP as your initial endpoint when setting up Talos—you need to bootstrap using the individual node IPs first.

Also, as the Talos docs warn: **don't use the VIP in your talosconfig endpoint**. Since the VIP depends on etcd and the Kubernetes API server being healthy, you won't be able to recover from certain failures if you're trying to manage Talos through the VIP.

## Configuration

Talos has built-in support for VIPs (powered by kube-vip under the hood), making the setup quite straightforward. I just needed to add the configuration to my `talconfig.yaml`.

### Choosing the VIP Address

My control plane nodes were using:
- allyrion: `10.4.0.10`
- bettley: `10.4.0.11`
- cargyll: `10.4.0.12`

I picked `10.4.0.9` for the VIP—an unused IP in the same subnet, which my DHCP server wouldn't assign.

### Interface Name Gotcha

The first attempt didn't work. I initially configured the VIP on `eth0`... forgetting that Raspberry Pis running Talos use predictable interface names, so the primary interface is actually `end0`. Once I fixed that, everything worked perfectly.

Here's the final configuration in `talconfig.yaml`:

```yaml
additionalApiServerCertSans:
  - 10.4.0.9  # VIP address
  - 10.4.0.10
  - 10.4.0.11
  - 10.4.0.12
  - cp.k8s.goldentooth.net

controlPlane:
  patches:
    - |-
      machine:
        network:
          interfaces:
            - interface: end0  # Not eth0!
              dhcp: true
              vip:
                ip: 10.4.0.9
```

### DNS Update

I also updated the Terraform configuration to point the DNS record to just the VIP instead of round-robin:

```terraform
resource "aws_route53_record" "k8s_control_plane" {
  zone_id = local.zone_id
  name    = "cp.k8s.goldentooth.net"
  type    = "A"
  ttl     = local.default_ttl

  records = [
    "10.4.0.9", # kube-vip VIP for control plane HA
  ]
}
```

## Deployment

After updating the configuration:

1. Applied the Terraform changes to update DNS
2. Regenerated the Talos machine configs: `talhelper genconfig`
3. Applied the new configs to each control plane:
   ```bash
   talosctl apply-config -n 10.4.0.10 -f clusterconfig/goldentooth-allyrion.yaml
   talosctl apply-config -n 10.4.0.11 -f clusterconfig/goldentooth-bettley.yaml
   talosctl apply-config -n 10.4.0.12 -f clusterconfig/goldentooth-cargyll.yaml
   ```

Within seconds, the VIP came alive:

```bash
$ ping -c 3 10.4.0.9
PING 10.4.0.9 (10.4.0.9): 56 data bytes
64 bytes from 10.4.0.9: icmp_seq=0 ttl=63 time=3.618 ms
64 bytes from 10.4.0.9: icmp_seq=1 ttl=63 time=2.714 ms
64 bytes from 10.4.0.9: icmp_seq=2 ttl=63 time=3.117 ms

$ kubectl get nodes
NAME        STATUS   ROLES           AGE   VERSION
allyrion    Ready    control-plane   19d   v1.34.0
bettley     Ready    control-plane   19d   v1.34.0
cargyll     Ready    control-plane   19d   v1.34.0
...
```

## Checking VIP Ownership

You can see which node currently owns the VIP by checking the network addresses:

```bash
$ talosctl -n 10.4.0.10 get addresses | grep "10.4.0.9"
10.4.0.10   network     AddressStatus   end0/10.4.0.9/32         10.4.0.9/32    end0
```

In this case, allyrion (`10.4.0.10`) is the current owner. If it fails, one of the other control planes will take over automatically.
