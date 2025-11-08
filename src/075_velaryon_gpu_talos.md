# Velaryon Returns: GPU Support in Talos

After migrating the Goldentooth cluster to Talos Linux (see [070_talos](./070_talos.md)), I had one major piece left to bring over: Velaryon, the x86 GPU node that originally joined the cluster way back in [046_new_server](./046_new_server.md).

Back then, I'd added this RTX 2070 Super-equipped machine to run GPU-heavy workloads that would be impossible on the Raspberry Pis. But with the switch to Talos, I needed to figure out how to properly integrate it into the new cluster with full GPU support.

## Why Talos for a GPU Node?

The old setup ran Ubuntu 24.04 with manually installed NVIDIA drivers and Kubernetes tooling. It worked, but it was a special snowflake that didn't match the rest of the cluster's configuration-as-code approach.

With Talos, I could:
- Use the same declarative configuration pattern as the Pi nodes
- Leverage Talos's system extension mechanism for NVIDIA drivers
- Get immutable infrastructure even for the GPU node
- Manage everything through the same `talconfig.yaml` file

## Talos Image Factory for NVIDIA

Talos doesn't ship with NVIDIA drivers baked in (for good reason—most nodes don't need them). Instead, you use the [Image Factory](https://factory.talos.dev) to build a custom Talos image with the NVIDIA system extensions included.

The process is straightforward:
1. Select the Talos version (v1.11.1 in my case)
2. Choose the system extensions needed:
   - `siderolabs/nvidia-driver` - The NVIDIA kernel modules
   - `siderolabs/nvidia-container-toolkit` - Container runtime integration
3. Get a custom image URL to use for installation

The Image Factory generates a unique URL that I added to Velaryon's configuration in `talconfig.yaml`:

```yaml
- hostname: velaryon
  ipAddress: 10.4.0.30
  installDisk: /dev/nvme0n1
  controlPlane: false
  nodeLabels:
    role: 'gpu'
    slot: 'X'
  talosImageURL: factory.talos.dev/metal-installer/af8eb82417d3deaa94d2ef19c3b590b0dac1b2549d0b9b35b3da2bc325de75f7
  patches:
    - |-
      machine:
        kernel:
          modules:
            - name: nvidia
            - name: nvidia_uvm
            - name: nvidia_drm
            - name: nvidia_modeset
```

The kernel modules patch ensures the NVIDIA drivers load at boot. The labels (`role: gpu`) make it easy to target GPU workloads to this specific node.

## Kubernetes RuntimeClass Configuration

Just having the NVIDIA drivers loaded isn't enough—Kubernetes needs to know how to use them. This is where `RuntimeClass` comes in.

A RuntimeClass tells Kubernetes to use a specific container runtime handler for pods. For NVIDIA GPUs, we need the `nvidia` runtime handler, which sets up the proper device access and library paths.

I created the RuntimeClass manifest in the GitOps repository:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia
```

This went into `/gitops/infrastructure/nvidia/` along with a dedicated namespace for GPU workloads. The namespace uses the `privileged` PodSecurity policy since GPU containers need device access that violates the standard `restricted` policy.

## Deployment

After updating the configurations:

1. Generated the custom Talos image and added it to talconfig.yaml
2. Installed Talos on Velaryon using the custom image
3. Applied the node configuration with kernel module patches
4. Committed the RuntimeClass manifests to the gitops repository
5. Let Flux reconcile the infrastructure changes

Within a few minutes, Velaryon was back in the cluster:

```bash
$ kubectl get nodes velaryon
NAME       STATUS   ROLES    AGE     VERSION
velaryon   Ready    <none>   36s     v1.34.0
```

## Verification

To verify the GPU was accessible, I ran a quick test using NVIDIA's CUDA container:

```bash
kubectl run nvidia-test \
  --namespace nvidia \
  --restart=Never \
  -ti --rm \
  --image nvcr.io/nvidia/cuda:12.2.0-base-ubuntu22.04 \
  --overrides '{"spec": {"runtimeClassName": "nvidia", "nodeSelector": {"role": "gpu"}}}' \
  -- nvidia-smi
```

**Important note**: The CUDA version in the container must match what the driver supports. Velaryon's driver (version 535.247.01) supports CUDA 12.2, so I used the `cuda:12.2.0-base` image rather than newer versions.

The test succeeded beautifully:

```
Sat Nov  8 23:28:00 2025
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.247.01             Driver Version: 535.247.01   CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|=========================================+======================+======================|
|   0  NVIDIA GeForce RTX 2070 ...    On  | 00000000:2D:00.0 Off |                  N/A |
|  0%   45C    P8              19W / 215W |      1MiB /  8192MiB |      0%      Default |
+-----------------------------------------+----------------------+----------------------+
```

Perfect! The RTX 2070 Super is ready for GPU workloads.
