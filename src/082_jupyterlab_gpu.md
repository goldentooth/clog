# JupyterLab with GPU: The ML Workbench

After getting Velaryon's GPU working in Talos (see [075_velaryon_gpu_talos](./075_velaryon_gpu_talos.md)), I had the hardware foundation for machine learning workloads. But running `nvidia-smi` in a test pod is a far cry from actually doing ML work. I wanted a proper development environment—something where I could fire up a notebook, import PyTorch, and start experimenting with neural networks.

The goal: JupyterLab running on Kubernetes with full GPU access, persistent storage for notebooks, and authentication that doesn't change every time the pod restarts.

## The Architecture

The ML workbench needed several pieces working together:

```
┌─────────────────────────────────────────────────────────────────┐
│                        JupyterLab Pod                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  cschranz/gpu-jupyter:v1.5_cuda-12.0_ubuntu-22.04       │    │
│  │  - PyTorch 2.1.2 + CUDA                                 │    │
│  │  - TensorFlow                                           │    │
│  │  - JupyterLab server                                    │    │
│  └─────────────────────────────────────────────────────────┘    │
│              │                              │                   │
│              ▼                              ▼                   │
│     ┌────────────────┐            ┌─────────────────┐           │
│     │ nvidia.com/gpu │            │ PVC (local-path)│           │
│     │     = 1        │            │  50Gi NVMe      │           │
│     └────────────────┘            └─────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │  NVIDIA Device Plugin │
        │  (advertises GPU to   │
        │   K8s scheduler)      │
        └───────────────────────┘
```

## Missing Piece #1: NVIDIA Device Plugin

Velaryon had the GPU drivers and the RuntimeClass configured, but Kubernetes didn't actually know a GPU existed. Running `kubectl get node velaryon -o jsonpath='{.status.capacity}'` showed CPU and memory, but no `nvidia.com/gpu` resource.

The NVIDIA Device Plugin is a DaemonSet that discovers GPUs on nodes and advertises them to the Kubernetes scheduler. Without it, you can't request `nvidia.com/gpu: 1` in your pod spec — Kubernetes has no idea what that means.

I deployed it via Helm in the GitOps repo:

```yaml
# gitops/infrastructure/nvidia/release.yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: nvidia-device-plugin
  namespace: flux-system
spec:
  chart:
    spec:
      chart: nvidia-device-plugin
      version: "0.18.0"
      sourceRef:
        kind: HelmRepository
        name: nvidia-device-plugin
  values:
    runtimeClassName: nvidia
    nodeSelector:
      role: gpu
    tolerations:
      - key: gpu
        operator: Equal
        value: "true"
        effect: NoSchedule
```

The chart has a default affinity that looks for nodes with NVIDIA hardware via the `nvidia.com/gpu.present` label. Since I was using a custom `role: gpu` label, I had to add the expected label to Velaryon's Talos config:

```yaml
# cluster/talconfig.yaml
- hostname: velaryon
  nodeLabels:
    role: 'gpu'
    nvidia.com/gpu.present: 'true'
```

After applying the config with `talosctl apply-config`, the Device Plugin scheduled and suddenly:

```json
{
  "cpu": "24",
  "memory": "32786032Ki",
  "nvidia.com/gpu": "1"
}
```

The GPU exists to Kubernetes now.

## Missing Piece #2: SeaweedFS CSI Driver

I wanted notebooks to persist across pod restarts. SeaweedFS was already running in the cluster, but there was no CSI driver to provision PersistentVolumeClaims from it.

The SeaweedFS CSI driver translates Kubernetes storage concepts (PVC, StorageClass) into SeaweedFS Filer operations:

```yaml
# gitops/infrastructure/seaweedfs/csi/release.yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: seaweedfs-csi-driver
spec:
  chart:
    spec:
      chart: seaweedfs-csi-driver
      version: "0.2.3"  # Note: Chart version, not app version!
  values:
    seaweedfsFiler: "goldentooth-storage-filer.seaweedfs:8888"
    storageClassName: seaweedfs
    isDefaultStorageClass: false
```

## The RBAC Rabbit Hole

When I tried to provision a PVC using `local-path` storage (for faster NVMe access), the provisioner failed with:

```
nodes "velaryon" is forbidden: User "system:serviceaccount:local-path-storage:local-path-provisioner-service-account" cannot get resource "nodes"
```

Turns out I had two local-path provisioners—one for regular storage and one for USB SSDs—and they were fighting over the same ClusterRoleBinding. The USB provisioner deployed second and overwrote the binding with its own namespace.

The fix was giving each provisioner its own explicitly-named ClusterRoleBinding:

```yaml
# local-path-provisioner/rbac-fix.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: local-path-provisioner-bind-storage
roleRef:
  kind: ClusterRole
  name: local-path-provisioner-role
subjects:
  - kind: ServiceAccount
    name: local-path-provisioner-service-account
    namespace: local-path-storage
```

Same for the USB provisioner with `-bind-usb`. Both bind to the same ClusterRole (the permissions are identical), they just don't clobber each other's bindings anymore.

## JupyterLab Deployment

With GPU and storage sorted, the JupyterLab deployment itself was straightforward:

```yaml
# gitops/apps/jupyterlab/deployment.yaml
spec:
  template:
    spec:
      runtimeClassName: nvidia
      nodeSelector:
        role: gpu
      tolerations:
        - key: gpu
          operator: Equal
          value: "true"
          effect: NoSchedule
      containers:
        - name: jupyterlab
          image: cschranz/gpu-jupyter:v1.5_cuda-12.0_ubuntu-22.04
          env:
            - name: JUPYTER_TOKEN
              valueFrom:
                secretKeyRef:
                  name: jupyterlab-auth
                  key: token
          resources:
            limits:
              nvidia.com/gpu: "1"
          volumeMounts:
            - name: workspace
              mountPath: /home/jovyan/work
      volumes:
        - name: workspace
          persistentVolumeClaim:
            claimName: jupyterlab-workspace
```

The key pieces:
- `runtimeClassName: nvidia` — Uses Talos's NVIDIA container runtime
- `nodeSelector` and `tolerations` — Ensures scheduling on Velaryon
- `nvidia.com/gpu: 1` — Requests the GPU from the Device Plugin
- SOPS-encrypted secret for the authentication token

The `cschranz/gpu-jupyter` image is a community project that combines Jupyter Docker Stacks with CUDA support. It comes with PyTorch, TensorFlow, and common ML libraries pre-installed. The image is ~8GB, so first pull takes a while.

## The Moment of Truth

After pushing all the configs and waiting for Flux to reconcile (and the massive image to pull), I opened `http://10.4.11.6` in my browser, entered my password, and ran:

```python
import torch
print(f"PyTorch version: {torch.__version__}")
print(f"CUDA available: {torch.cuda.is_available()}")
print(f"GPU: {torch.cuda.get_device_name(0)}")
```

Output:
```
PyTorch version: 2.1.2+cu121
CUDA available: True
GPU: NVIDIA GeForce RTX 2070 SUPER
```

And a quick matrix multiply to prove it actually works:

```python
x = torch.randn(1000, 1000, device='cuda')
y = torch.randn(1000, 1000, device='cuda')
z = torch.matmul(x, y)
print(f"Matrix multiply on GPU: {z.shape}")
```

![JupyterLab GPU Test](./images/082_jupyterlab_gpu_test.png)

8.4GB of VRAM ready for neural network experiments. Not bad for a Raspberry Pi cluster's sidekick.

The infrastructure is ready for Karpathy's nanoGPT and similar educational ML projects. The RTX 2070 Super's 8GB VRAM can handle GPT-2 sized models comfortably—plenty for learning transformer architectures from scratch.

Time to train some tiny language models.
