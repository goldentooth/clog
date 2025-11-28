# GPU Gaming Containers: The Quest Begins

After getting JupyterLab working with GPU support (see [082_jupyterlab_gpu](./082_jupyterlab_gpu.md)), I had a taste of what GPU workloads could do in Kubernetes. Neural networks are cool, but you know what would be cooler? Playing Baldur's Gate 3 streamed from a containerized Steam instance running on my cluster.

The plan: Use Packer to build a gaming container image with Steam and Proton, run it on velaryon's RTX 2070 Super, and stream games via Steam Remote Play. This is definitely overkill for playing video games, but that's never stopped me before.

## The Architecture: CUDA vs OpenGL

Before diving in, I needed to understand what makes gaming different from machine learning workloads. JupyterLab uses CUDA—NVIDIA's framework for general-purpose parallel computing. You write kernels that run math operations across thousands of GPU cores. Matrix multiplication, neural network training, that sort of thing.

Gaming uses OpenGL (or Vulkan), which is specifically designed for 3D graphics rendering. It's a fixed pipeline:

```
Vertices → Vertex Shader → Rasterization → Fragment Shader → Pixels
```

Both use the same GPU hardware, but they're different programming models. CUDA is "here's data and a function, run it everywhere." OpenGL is "here's a 3D scene, transform and render it through these specific stages."

The RTX 2070 Super has both capabilities. JupyterLab already proved CUDA works. Now I needed to verify the graphics path worked too.

## Stage 1: Verifying the Foundation

First, I checked that the NVIDIA device plugin was still running:

```bash
$ kubectl get pods -n nvidia
NAME                                READY   STATUS    RESTARTS   AGE
nvidia-nvidia-device-plugin-wflzh   2/2     Running   0          3d23h
```

Good. The device plugin is what advertises GPU resources to Kubernetes. Without it, you can't request `nvidia.com/gpu: 1` in your pod spec.

Next, I verified velaryon was advertising GPU capacity:

```bash
$ kubectl get node velaryon -o jsonpath='{.status.capacity}' | jq
{
  "cpu": "24",
  "memory": "32786032Ki",
  "nvidia.com/gpu": "4",
  ...
}
```

Four GPUs! Well, one physical GPU time-sliced into four virtual ones. The time-slicing config from the JupyterLab setup advertises 1 physical RTX 2070 Super as 4 resources via context-switching. JupyterLab is using one slot, leaving three free.

## Testing CUDA Access

I created a simple test pod to verify CUDA still worked:

```yaml
# gpu-test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  runtimeClassName: nvidia  # Use NVIDIA container runtime
  nodeSelector:
    role: gpu
  tolerations:
    - key: gpu
      operator: Equal
      value: "true"
      effect: NoSchedule
  containers:
    - name: gpu-test
      image: nvidia/cuda:12.0.0-base-ubuntu22.04
      command: ["nvidia-smi"]
      resources:
        limits:
          nvidia.com/gpu: "1"
```

The key pieces:
- `runtimeClassName: nvidia` - Tells Kubernetes to use Talos's NVIDIA container runtime (which mounts GPU device files and driver libraries)
- `resources.limits.nvidia.com/gpu: 1` - Requests one GPU from the device plugin
- `nodeSelector` and `tolerations` - Ensures it schedules on velaryon

```bash
$ kubectl apply -f gpu-test-pod.yaml
$ kubectl logs gpu-test
Thu Nov 27 16:58:24 2025
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.247.01             Driver Version: 535.247.01   CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|=========================================+======================+======================|
|   0  NVIDIA GeForce RTX 2070 ...    On  | 00000000:2D:00.0 Off |                  N/A |
|  0%   46C    P8              19W / 215W |    749MiB /  8192MiB |      0%      Default |
+---------------------------------------------------------------------------------------+
```

Perfect. CUDA access works. The container sees the GPU, the driver, everything. The 749MiB of used VRAM is JupyterLab sitting idle.

## Testing OpenGL: The llvmpipe Problem

CUDA working doesn't guarantee OpenGL works. Time to test graphics rendering. I created a test that runs `glxgears` (a simple OpenGL demo) in a container:

```yaml
# gpu-opengl-test-pod.yaml (first attempt)
apiVersion: v1
kind: Pod
metadata:
  name: gpu-opengl-test
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
    - name: opengl-test
      image: nvidia/opengl:1.2-glvnd-runtime-ubuntu22.04
      command:
        - /bin/bash
        - -c
        - |
          apt-get update -qq
          apt-get install -y -qq mesa-utils xvfb
          Xvfb :99 -screen 0 1024x768x24 &
          export DISPLAY=:99
          sleep 2
          glxinfo | grep -E "OpenGL renderer|OpenGL version"
          timeout 5s glxgears -info
      resources:
        limits:
          nvidia.com/gpu: "1"
      env:
        - name: NVIDIA_VISIBLE_DEVICES
          value: "all"
        - name: NVIDIA_DRIVER_CAPABILITIES
          value: "all"
```

The script:
1. Installs `mesa-utils` (OpenGL testing tools) and `xvfb` (virtual X server)
2. Starts Xvfb on display :99 (OpenGL needs an X server to create a rendering context)
3. Runs `glxinfo` to see what OpenGL renderer is being used

The result:

```
OpenGL renderer string: llvmpipe (LLVM 15.0.7, 256 bits)
OpenGL version string: 4.5 (Compatibility Profile) Mesa 23.2.1
```

**llvmpipe**. That's Mesa's CPU-based software renderer. OpenGL was rendering on the CPU, not the GPU. Games would run at maybe 2 FPS with this.

## The Debugging Journey

This was frustrating. `nvidia-smi` worked perfectly, proving the GPU was accessible. But OpenGL couldn't see it. Something was broken in the graphics path specifically.

I deployed a long-running debug pod to investigate:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-debug
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
    - name: debug
      image: nvidia/opengl:1.2-glvnd-runtime-ubuntu22.04
      command: ["sleep", "infinity"]
      resources:
        limits:
          nvidia.com/gpu: "1"
      env:
        - name: NVIDIA_VISIBLE_DEVICES
          value: "all"
        - name: NVIDIA_DRIVER_CAPABILITIES
          value: "all"
```

Then I exec'd in and started poking around:

```bash
$ kubectl exec -it gpu-debug -- bash

# Check if NVIDIA libraries are mounted
root@gpu-debug:/# ls -la /usr/lib/x86_64-linux-gnu/libnvidia* | head -20
lrwxrwxrwx. 1 root root  33 Nov 27 17:28 /usr/lib/x86_64-linux-gnu/libnvidia-allocator.so.1 -> libnvidia-allocator.so.535.247.01
-rwxr-xr-x. 1 root root 160552 Jun  1  2019 /usr/lib/x86_64-linux-gnu/libnvidia-allocator.so.535.247.01
...
-rwxr-xr-x. 1 root root 45959040 Jun  1  2019 /usr/lib/x86_64-linux-gnu/libnvidia-glcore.so.535.247.01
-rwxr-xr-x. 1 root root 656472 Jun  1  2019 /usr/lib/x86_64-linux-gnu/libnvidia-glsi.so.535.247.01
```

The NVIDIA graphics libraries were there! `libnvidia-glcore`, `libnvidia-glsi`, all the OpenGL stuff. The NVIDIA runtime was doing its job and mounting the driver libraries.

```bash
# Check GLX libraries
root@gpu-debug:/# ls -la /usr/lib/x86_64-linux-gnu/libGL*
lrwxrwxrwx. 1 root root   14 Jan  4  2022 /usr/lib/x86_64-linux-gnu/libGL.so.1 -> libGL.so.1.7.0
-rw-r--r--. 1 root root 543056 Jan  4  2022 /usr/lib/x86_64-linux-gnu/libGL.so.1.7.0
...
lrwxrwxrwx. 1 root root   27 Nov 27 17:28 /usr/lib/x86_64-linux-gnu/libGLX_nvidia.so.0 -> libGLX_nvidia.so.535.247.01
-rwxr-xr-x. 1 root root 1195552 Jun  1  2019 /usr/lib/x86_64-linux-gnu/libGLX_nvidia.so.535.247.01
lrwxrwxrwx. 1 root root   20 Dec 15  2022 /usr/lib/x86_64-linux-gnu/libGLX_mesa.so.0 -> libGLX_mesa.so.0.0.0
-rw-r--r--. 1 root root 459672 Dec 15  2022 /usr/lib/x86_64-linux-gnu/libGLX_mesa.so.0.0.0
```

Interesting. Both NVIDIA and Mesa GLX libraries existed. `libGLX_nvidia.so` was the NVIDIA implementation, `libGLX_mesa.so` was the software renderer. OpenGL was choosing Mesa.

```bash
# Check ICD (Installable Client Driver) configs
root@gpu-debug:/# ls -la /usr/share/glvnd/egl_vendor.d/
total 5
drwxr-xr-x. 1 root root  28 May 17  2023 .
drwxr-xr-x. 1 root root  26 May 17  2023 ..
-rw-r--r--. 1 root root 107 Jun  1  2019 10_nvidia.json
-rw-r--r--. 1 root root 105 Dec 15  2022 50_mesa.json
```

ICD config files tell GLVND (the OpenGL loader) which vendor libraries to use. The files are numbered—lower numbers have higher priority. `10_nvidia.json` should be preferred over `50_mesa.json`.

```bash
root@gpu-debug:/# cat /usr/share/glvnd/egl_vendor.d/10_nvidia.json
{
    "file_format_version" : "1.0.0",
    "ICD" : {
        "library_path" : "libEGL_nvidia.so.0"
    }
}
```

Wait. This ICD config is for **EGL** (`libEGL_nvidia.so.0`), not GLX.

## EGL vs GLX: The Missing Link

EGL and GLX are two different OpenGL windowing systems:
- **GLX** (GL + X11): Traditional Linux OpenGL, works with X11 servers like Xvfb
- **EGL** (Embedded GL): Modern, platform-agnostic OpenGL for Wayland or headless contexts

When I ran `glxinfo`, I was testing **GLX**. But the NVIDIA ICD config only told GLVND about **EGL** libraries. There was no `/usr/share/glvnd/glx_vendor.d/` directory with GLX configs.

I tried setting an environment variable that explicitly tells GLVND to use NVIDIA for GLX:

```bash
root@gpu-debug:/# export __GLX_VENDOR_LIBRARY_NAME=nvidia
root@gpu-debug:/# glxinfo | grep "OpenGL renderer"
OpenGL renderer string: NVIDIA GeForce RTX 2070 SUPER/PCIe/SSE2
```

**There it is.** With `__GLX_VENDOR_LIBRARY_NAME=nvidia`, GLVND used the NVIDIA libraries instead of Mesa. GPU-accelerated rendering worked.

## The Fix

The solution was adding that environment variable to the pod spec:

```yaml
env:
  - name: NVIDIA_VISIBLE_DEVICES
    value: "all"
  - name: NVIDIA_DRIVER_CAPABILITIES
    value: "all"
  # Tell GLVND to use NVIDIA for GLX (not Mesa)
  - name: __GLX_VENDOR_LIBRARY_NAME
    value: "nvidia"
```

After updating the test pod and rerunning:

```
OpenGL renderer string: NVIDIA GeForce RTX 2070 SUPER/PCIe/SSE2
OpenGL version string: 4.6.0 NVIDIA 535.247.01
GL_VENDOR = NVIDIA Corporation
```

Perfect. OpenGL 4.6 support with full GPU acceleration. More than enough for modern games.

## What I Learned

**CUDA vs OpenGL**: CUDA is for general parallel compute (neural networks, matrix math). OpenGL is for 3D graphics rendering (games, visualization). Both use the same GPU hardware but through different programming models.

**Container Runtime Classes**: The `runtimeClassName: nvidia` tells Kubernetes to use Talos's NVIDIA container runtime, which mounts GPU device files (`/dev/nvidia*`) and driver libraries into the container. Without this, containers can't access the GPU even if the device plugin allocated a GPU resource.

**NVIDIA Environment Variables**:
- `NVIDIA_VISIBLE_DEVICES=all` - Makes all GPUs visible to the container
- `NVIDIA_DRIVER_CAPABILITIES=all` - Enables graphics, compute, video encoding/decoding, etc.
- `__GLX_VENDOR_LIBRARY_NAME=nvidia` - Tells GLVND to use NVIDIA's GLX implementation instead of Mesa

**GLVND Vendor Dispatch**: GLVND (GL Vendor Neutral Dispatch) is the modern OpenGL loader that supports multiple GPU vendors on the same system. It uses ICD config files to discover which vendor libraries to load. In containers, the EGL configs are present but the GLX configs are missing, so you need the `__GLX_VENDOR_LIBRARY_NAME` env var for explicit vendor selection.

**The Debugging Process**: When something doesn't work, trace the path systematically:
1. Verify the hardware is accessible (nvidia-smi)
2. Check if libraries are mounted (`ls /usr/lib/.../libnvidia*`)
3. Check ICD loader configs (`/usr/share/glvnd/`)
4. Use environment variables to force specific behavior
5. Test in an interactive shell (`kubectl exec -it`) before building automated solutions

## Next Steps

Stage 1 is complete: GPU-accelerated OpenGL rendering works in containers. The foundation is solid.

Stage 2 will be building a full GUI container with VNC so I can actually see and interact with applications. That means:
- Setting up a proper X server (not just Xvfb)
- Installing a lightweight window manager
- Configuring TigerVNC for remote access
- Making it accessible via Kubernetes Service/Ingress

Then Stage 3: Getting Steam and Proton running in that container.

This is going to be fun. Or frustrating. Probably both.
