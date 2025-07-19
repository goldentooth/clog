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
- **Workers (9 nodes)**: erenford, fenn, gardener, harlton, inchfield, jast, karstark, lipps, velaryon (GPU)

This brings us to a total of 12 nodes in the Kubernetes cluster, matching the full complement of our Raspberry Pi bramble plus the x86 GPU node.

## GPU Node Configuration

[Velaryon](./046_new_server.md), my x86 GPU node, required special configuration to ensure GPU workloads are only scheduled intentionally:

### Hardware Specifications
- **GPU**: NVIDIA GeForce RTX 2070 (8GB VRAM)
- **CPU**: 24 cores (x86_64)
- **Memory**: 32GB RAM
- **Architecture**: amd64

### Kubernetes Configuration
The node is configured with:
- **Label**: `gpu=true` for workload targeting
- **Taint**: `gpu=true:NoSchedule` to prevent accidental scheduling
- **Architecture**: `arch=amd64` for x86-specific workloads

### Scheduling Requirements
To schedule workloads on Velaryon, pods must include:
```yaml
tolerations:
- key: gpu
  operator: Equal
  value: "true"
  effect: NoSchedule
nodeSelector:
  gpu: "true"
```

This ensures that only workloads explicitly designed for GPU execution can access the expensive GPU resources, following the same intentional scheduling pattern used with Nomad.

## GPU Resource Detection Challenge

While the taint-based scheduling was working correctly, getting Kubernetes to actually detect and expose the GPU resources proved more challenging. The NVIDIA device plugin is responsible for discovering GPUs and advertising them as `nvidia.com/gpu` resources that pods can request.

### Initial Problem

The device plugin was failing with the error:
```
E0719 16:20:41.050191       1 factory.go:115] Incompatible platform detected
E0719 16:20:41.050193       1 factory.go:116] If this is a GPU node, did you configure the NVIDIA Container Toolkit?
```

Despite having installed the NVIDIA Container Toolkit and configuring containerd, the device plugin couldn't detect the NVML library from within its container environment.

### The Root Cause

The issue was that the device plugin container couldn't access:
1. **NVIDIA Management Library**: `libnvidia-ml.so.1` needed for GPU discovery
2. **Device files**: `/dev/nvidia*` required for direct GPU communication
3. **Proper privileges**: Needed to interact with kernel-level GPU drivers

### The Solution

After several iterations, the working configuration required:

**Library Access**:
```yaml
volumeMounts:
- name: nvidia-ml-lib
  mountPath: /usr/lib/x86_64-linux-gnu/libnvidia-ml.so.1
  readOnly: true
- name: nvidia-ml-actual
  mountPath: /usr/lib/x86_64-linux-gnu/libnvidia-ml.so.575.51.03
  readOnly: true
```

**Device Access**:
```yaml
volumeMounts:
- name: dev
  mountPath: /dev
volumes:
- name: dev
  hostPath:
    path: /dev
```

**Container Privileges**:
```yaml
securityContext:
  privileged: true
```

### Verification

Once properly configured, the device plugin successfully reported:
```
I0719 16:56:06.462937       1 server.go:165] Starting GRPC server for 'nvidia.com/gpu'
I0719 16:56:06.463631       1 server.go:117] Starting to serve 'nvidia.com/gpu' on /var/lib/kubelet/device-plugins/nvidia-gpu.sock
I0719 16:56:06.465420       1 server.go:125] Registered device plugin for 'nvidia.com/gpu' with Kubelet
```

The GPU resource then appeared in the node's capacity:
```bash
kubectl get nodes -o json | jq '.items[] | select(.metadata.name=="velaryon") | .status.capacity'
```
```json
{
  "cpu": "24",
  "ephemeral-storage": "102626232Ki",
  "hugepages-1Gi": "0",
  "hugepages-2Mi": "0",
  "memory": "32803048Ki",
  "nvidia.com/gpu": "1",
  "pods": "110"
}
```

### Testing GPU Resource Allocation

To verify the system was working end-to-end, I created a test pod that:
- **Requests GPU resources**: `nvidia.com/gpu: 1`
- **Includes proper tolerations**: To bypass the `gpu=true:NoSchedule` taint
- **Targets the GPU node**: Using `gpu: "true"` node selector

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test-workload
spec:
  tolerations:
  - key: gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  nodeSelector:
    gpu: "true"
  containers:
  - name: gpu-test
    image: busybox
    command: ["sleep", "60"]
    resources:
      requests:
        nvidia.com/gpu: 1
      limits:
        nvidia.com/gpu: 1
```

The pod successfully scheduled and the node showed:
```
nvidia.com/gpu     1          1
```

This confirmed that GPU resource allocation tracking was working correctly.

## Final NVIDIA Device Plugin Configuration

For reference, here's the complete working NVIDIA device plugin DaemonSet configuration:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-daemonset
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: nvidia-device-plugin-ds
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: nvidia-device-plugin-ds
    spec:
      tolerations:
      - key: gpu
        operator: Equal
        value: "true"
        effect: NoSchedule
      nodeSelector:
        gpu: "true"
      priorityClassName: system-node-critical
      containers:
      - image: nvcr.io/nvidia/k8s-device-plugin:v0.14.5
        name: nvidia-device-plugin-ctr
        env:
        - name: FAIL_ON_INIT_ERROR
          value: "false"
        securityContext:
          privileged: true
        volumeMounts:
        - name: device-plugin
          mountPath: /var/lib/kubelet/device-plugins
        - name: dev
          mountPath: /dev
        - name: nvidia-ml-lib
          mountPath: /usr/lib/x86_64-linux-gnu/libnvidia-ml.so.1
          readOnly: true
        - name: nvidia-ml-actual
          mountPath: /usr/lib/x86_64-linux-gnu/libnvidia-ml.so.575.51.03
          readOnly: true
      volumes:
      - name: device-plugin
        hostPath:
          path: /var/lib/kubelet/device-plugins
      - name: dev
        hostPath:
          path: /dev
      - name: nvidia-ml-lib
        hostPath:
          path: /lib/x86_64-linux-gnu/libnvidia-ml.so.1
      - name: nvidia-ml-actual
        hostPath:
          path: /usr/lib/x86_64-linux-gnu/libnvidia-ml.so.575.51.03
```

Key aspects of this configuration:
- **Targeted deployment**: Only runs on nodes with `gpu: "true"` label
- **Taint tolerance**: Can schedule on nodes with `gpu=true:NoSchedule` taint
- **Privileged access**: Required for kernel-level GPU driver interaction
- **Library binding**: Specific mounts for NVIDIA ML library files
- **Device access**: Full `/dev` mount for GPU device communication
