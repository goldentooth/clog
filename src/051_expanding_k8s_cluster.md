# Expanding the Kubernetes Cluster

With the Goldentooth cluster continuing to evolve, it was time to bring two more nodes into the Kubernetes fold. Karstark and Lipps had been sitting on the sidelines, contributing to other cluster services but not yet participating in the Kubernetes orchestration layer.

## The Current State

Before the expansion, our Kubernetes cluster consisted of:

- **Control Plane (3 nodes)**: bettley, cargyll, dalt
- **Workers (7 nodes)**: erenford, fenn, gardener, harlton, inchfield, jast, velaryon

Karstark and Lipps were already fully integrated into the cluster infrastructure:
- Both were part of the Consul service mesh as clients
- Both were configured as Nomad clients for workload scheduling
- Both were included in other cluster services like Ray and Slurm

However, they weren't yet part of the Kubernetes cluster, which meant we were missing out on their compute capacity for containerized workloads.

## Installing Kubernetes Packages

The first step was to ensure both nodes had the necessary Kubernetes packages installed. Using the goldentooth CLI:

```bash
ansible-playbook -i inventory/hosts playbooks/install_k8s_packages.yaml --limit karstark,lipps
```

This playbook handled:
- Installing and configuring containerd as the container runtime
- Installing kubeadm, kubectl, and kubelet packages
- Setting up proper systemd cgroup configuration
- Enabling and starting the kubelet service

Both nodes successfully installed Kubernetes v1.32.7, which was slightly newer than the existing cluster nodes running v1.32.3.

## The Challenge: Certificate Issues

When attempting to use the standard `goldentooth bootstrap_k8s` command, we ran into certificate verification issues. The bootstrap process was timing out when trying to communicate with the Kubernetes API server.

The error manifested as:
```
tls: failed to verify certificate: x509: certificate signed by unknown authority
```

This is a common issue in clusters that have been running for a while (393 days in our case) and have undergone certificate rotations or updates.

## The Solution: Manual Join Process

Instead of relying on the automated bootstrap, I took a more direct approach:

1. **Generate a join token** from the control plane:
   ```bash
   goldentooth command_root bettley "kubeadm token create --print-join-command"
   ```

2. **Execute the join command** on both nodes:
   ```bash
   goldentooth command_root karstark,lipps "kubeadm join 10.4.0.10:6443 --token yi3zz8.qf0ziy9ce7nhnkjv --discovery-token-ca-cert-hash sha256:0d6c8981d10e407429e135db4350e6bb21382af57addd798daf6c3c5663ac964 --skip-phases=preflight"
   ```

The `--skip-phases=preflight` flag was key here, as it bypassed the problematic preflight checks while still allowing the nodes to join successfully.

## Verification

After the join process completed, both nodes appeared in the cluster:

```bash
goldentooth command_root bettley "kubectl get nodes"
```

```
NAME        STATUS   ROLES           AGE    VERSION
bettley     Ready    control-plane   393d   v1.32.3
cargyll     Ready    control-plane   393d   v1.32.3
dalt        Ready    control-plane   393d   v1.32.3
erenford    Ready    <none>          393d   v1.32.3
fenn        Ready    <none>          393d   v1.32.3
gardener    Ready    <none>          393d   v1.32.3
harlton     Ready    <none>          393d   v1.32.3
inchfield   Ready    <none>          393d   v1.32.3
jast        Ready    <none>          393d   v1.32.3
karstark    Ready    <none>          53s    v1.32.7
lipps       Ready    <none>          54s    v1.32.7
velaryon    Ready    <none>          52d    v1.32.5
```

Perfect! Both nodes transitioned from "NotReady" to "Ready" status within about a minute, indicating that the Calico CNI networking had successfully configured them.

## The New Topology

Our Kubernetes cluster now consists of:

- **Control Plane (3 nodes)**: bettley, cargyll, dalt
- **Workers (9 nodes)**: erenford, fenn, gardener, harlton, inchfield, jast, karstark, lipps, velaryon

This brings us to a total of 12 nodes in the Kubernetes cluster, matching the full complement of our Raspberry Pi bramble plus the x86 GPU node.

