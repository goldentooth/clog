# Joining Pi 5 Nodes: The Manual Bootstrap

The last entry about netboot ([089](./089_netboot.md)) ended with a cheerful "The 4 Pi 5 nodes need different firmware — the Pi 5 has a completely different boot architecture. That's a problem for future me." Well, future me showed up, looked at the problem, and decided to go a completely different direction.

The plan was always to run Talos on the Pi 5s, same as the rest of the bramble. The Pi 5 has NVMe storage, newer silicon, more RAM — it would be the storage tier, running Longhorn on those 932GB NVMe SSDs. Beautiful plan. One problem: it doesn't work.

## The Kernel Bug

Siderolabs ships an SBC overlay for the Raspberry Pi that includes kernel patches and device tree modifications for Talos compatibility. On the Pi 5, [there's a bug](https://github.com/siderolabs/sbc-raspberrypi/issues/82) where the Ethernet interface fails to initialize. The NIC just... doesn't come up. No link, no DHCP, nothing. The node boots into Talos and sits there uselessly.

I burned more time than I'd like to admit trying workarounds — different Talos versions, different overlay builds, netboot with custom kernel parameters. The bug is in the kernel's BCM2712 Ethernet driver as built for the SBC overlay, and nothing short of a kernel fix is going to help.

## The Pivot: Ubuntu Server

Fine. If Talos won't run on the Pi 5, Ubuntu will. Ubuntu 25.10 has working arm64 support for the Pi 5, including the Ethernet driver (because of course it does — it's not trying to be special). I flashed SD cards with the preinstalled server image, configured cloud-init for static IPs and SSH keys, and after some fiddling with XFS formatting on the NVMe drives, had four healthy Ubuntu nodes:

| Node     | IP        | NVMe  |
| -------- | --------- | ----- |
| manderly | 10.4.0.22 | 932GB |
| norcross | 10.4.0.23 | 932GB |
| oakheart | 10.4.0.24 | 932GB |
| payne    | 10.4.0.25 | 932GB |

Each with containerd, kubelet, and kubeadm installed from the Kubernetes apt repo. NVMe drives formatted XFS and mounted at `/var/lib/longhorn`. Ready to join the cluster.

## Attempt 1: kubeadm join (Three Flavors of Failure)

The obvious approach: `kubeadm join`. That's what it's for. You give it a token, a CA hash, and a control plane endpoint, and it handles the TLS bootstrap dance.

```bash
kubeadm join cp.k8s.goldentooth.net:6443 \
  --token pi5wrk.abcdef1234567890 \
  --discovery-token-ca-cert-hash sha256:0e7e249a...
```

This failed immediately. The token-based discovery mechanism expects to read the `cluster-info` ConfigMap from the `kube-public` namespace using anonymous authentication. Talos disables anonymous auth to the API server. No anonymous access means no reading `kube-public`, which means the discovery phase can't obtain the cluster CA to verify the API server's identity. Chicken, meet egg.

**Attempt 1b: File-based discovery.** kubeadm supports `--discovery-file` where you provide a kubeconfig with the CA already embedded, skipping the anonymous `kube-public` lookup:

```bash
kubeadm join cp.k8s.goldentooth.net:6443 \
  --discovery-file /etc/kubernetes/discovery.yaml
```

This got further — the TLS handshake succeeded, the bootstrap token authenticated — and then kubeadm tried to read the `kubeadm-config` ConfigMap from `kube-system`. Talos doesn't create this ConfigMap. It's a kubeadm-specific artifact that contains the `ClusterConfiguration` — things like the API server address, networking CIDRs, certificate SANs. Talos manages all of that through its own machine config and doesn't need kubeadm's opinion about it.

The error was a clean `Forbidden` — the bootstrap token doesn't have RBAC access to read arbitrary ConfigMaps in `kube-system`, and even if it did, the ConfigMap doesn't exist.

I briefly considered creating a fake `kubeadm-config` ConfigMap with the right fields to satisfy kubeadm's checks. Then I had a better idea.

## The Manual Bootstrap

kubeadm is, at its core, a convenience wrapper around the kubelet's TLS bootstrap protocol. The kubelet has built-in support for bootstrapping itself into a cluster using a bootstrap token. kubeadm just automates the config file generation and CSR approval setup. But you can do all of that by hand.

The protocol is:
1. kubelet starts with a bootstrap kubeconfig (token-based auth)
2. kubelet generates a client key pair and submits a CertificateSigningRequest
3. The CSR gets auto-approved (if RBAC is set up for it)
4. kubelet receives a signed client certificate, writes a permanent kubeconfig
5. kubelet is now a full cluster member

I needed four files on each node:

### 1. CA Certificate

This one's easy — it's in the `kube-root-ca.crt` ConfigMap that every namespace gets automatically:

```bash
kubectl get configmap kube-root-ca.crt -n default -o jsonpath='{.data.ca\.crt}'
```

Goes to `/etc/kubernetes/pki/ca.crt`. The cluster CA is an EC key (P-256, ecdsa-with-SHA256), which I discovered the hard way when `openssl rsa -pubin` refused to parse it. `openssl pkey -pubin` works for any key type. Filed that one away.

### 2. Bootstrap Kubeconfig

A kubeconfig that authenticates with the bootstrap token:

```yaml
apiVersion: v1
kind: Config
clusters:
  - cluster:
      certificate-authority-data: <base64 CA cert>
      server: https://cp.k8s.goldentooth.net:6443
    name: goldentooth
contexts:
  - context:
      cluster: goldentooth
      user: kubelet-bootstrap
    name: bootstrap
current-context: bootstrap
users:
  - name: kubelet-bootstrap
    user:
      token: pi5wrk.abcdef1234567890
```

Goes to `/etc/kubernetes/bootstrap-kubelet.conf`. The bootstrap token was already created and patched to the `system:bootstrappers:nodes` group, with RBAC allowing that group to create CSRs and auto-approve node client certificates.

### 3. Kubelet Configuration

This one took some archaeology. I needed a `KubeletConfiguration` that matched what the rest of the cluster expected. The Talos nodes don't have a file you can just cat — Talos generates the kubelet config on the fly from its machine config. But kubelet exposes its running config via the node proxy API:

```bash
kubectl get --raw /api/v1/nodes/dalt/proxy/configz
```

This returns the live `KubeletConfiguration` from a running Talos worker. I used it as a reference and adapted for Ubuntu:

- **`cgroupDriver: systemd`** instead of `cgroupfs` — Ubuntu uses systemd cgroups, Talos uses its own cgroup management
- **`protectKernelDefaults: false`** — Talos sets `true` because it controls all kernel parameters. Ubuntu's defaults don't satisfy the kubelet's kernel parameter checks
- **Standard resolv.conf** — Talos uses `/system/resolved/resolv.conf`, Ubuntu uses `/run/systemd/resolve/resolv.conf`
- **No custom cgroup paths** — Talos sets `kubeletCgroups` and `systemCgroups` explicitly, Ubuntu lets systemd handle it

The rest carried over: `clusterDNS: [10.96.0.10]`, `clusterDomain: cluster.local`, `rotateCertificates: true`, `tlsMinVersion: VersionTLS13`, `seccompDefault: true`, pod CIDR `10.244.0.0/16`, service CIDR `10.96.0.0/12`, max pods 110.

Goes to `/var/lib/kubelet/config.yaml`.

### 4. Systemd Drop-in

The kubelet package from the Kubernetes apt repo ships a bare systemd unit — just `ExecStart=/usr/bin/kubelet` with no arguments. It expects a drop-in to provide the actual flags:

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/kubelet \
  --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf \
  --kubeconfig=/etc/kubernetes/kubelet.conf \
  --config=/var/lib/kubelet/config.yaml \
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --node-labels=node.kubernetes.io/disk-type=nvme
```

The `ExecStart=` (blank) line is important — systemd requires you to clear the directive before overriding it in a drop-in. Without that, you get both ExecStart lines and systemd refuses to start the unit.

The kubeadm apt package also ships its own drop-in at `/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf` that references `$KUBELET_KUBEADM_ARGS` and other environment variables that don't exist in our setup. That file had to be removed before the kubelet would start cleanly.

## First Blood: Manderly

With all four files in place, I started kubelet on Manderly and watched the logs. The TLS bootstrap worked on the first try:

```
I0312 kubelet_certificate_manager.go:263] "Certificate rotation is enabled"
I0312 certificate_manager.go:454] "Rotating certificates"
I0312 certificate_manager.go:497] "Certificate approved, waiting to be issued"
```

The bootstrap token authenticated, the CSR was submitted and auto-approved, and kubelet received a signed client certificate. Node appeared in the cluster as `NotReady`:

```
$ kubectl get node manderly
NAME       STATUS     ROLES    AGE   VERSION
manderly   NotReady   <none>   12s   v1.34.5
```

`NotReady` because the CNI wasn't running yet. Cilium is a DaemonSet — it should schedule automatically on any new node. And it did schedule. It just didn't start.

## The localhost:7445 Problem

Cilium's init containers were stuck at `Init:0/5`. The logs revealed the issue:

```
level=info msg="Establishing connection to apiserver" host="https://localhost:7445"
```

The Cilium DaemonSet has hardcoded environment variables:

```yaml
env:
  - name: KUBERNETES_SERVICE_HOST
    value: "localhost"
  - name: KUBERNETES_SERVICE_PORT
    value: "7445"
```

This is a Talos-ism. Talos runs KubePrism, a local API server proxy on every node at `localhost:7445` that forwards to the actual control plane. This means pods don't need to know the real API server address — they just talk to localhost. It's clever and it works great on Talos.

Ubuntu doesn't have KubePrism. There's nothing listening on `localhost:7445`. Cilium starts, tries to connect to the API server, gets connection refused, and sits in init forever.

I considered modifying the Cilium DaemonSet to use the real API server address, but that would break every Talos node. I considered running a separate Cilium DaemonSet for Ubuntu nodes with a different config, but that's a maintenance nightmare.

The simplest solution: give Ubuntu nodes their own `localhost:7445` proxy.

```bash
apt-get install -y socat
```

Then a systemd service:

```ini
[Unit]
Description=Kubernetes API Server Proxy (socat)
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/socat TCP-LISTEN:7445,bind=127.0.0.1,reuseaddr,fork TCP:10.4.0.9:6443
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

socat listens on `localhost:7445` and forwards every connection to the real control plane VIP at `10.4.0.9:6443`. It's not as sophisticated as KubePrism (no health checking, no endpoint rotation), but the VIP handles failover at the MetalLB level, so it doesn't need to be.

After enabling the proxy service, I deleted the stuck Cilium pods. The DaemonSet scheduled new ones, they connected through the socat proxy, and Cilium initialized:

```
$ kubectl get node manderly
NAME       STATUS   ROLES    AGE     VERSION
manderly   Ready    <none>   4m33s   v1.34.5
```

Ready. First Pi 5 in the cluster.

## Rolling Out to the Fleet

With the process proven on manderly, the remaining three nodes were mechanical. The exact same files — CA cert, bootstrap kubeconfig, kubelet config, systemd drop-in — apply to all nodes because the bootstrap token is shared and the kubelet config is node-agnostic. Each node generates its own client certificate during TLS bootstrap.

For each of norcross, oakheart, and payne:

1. SCP the four config files
2. Install socat, place files in the right paths
3. Remove the kubeadm drop-in
4. Create and enable the API proxy service
5. Start kubelet
6. Delete the stuck Cilium pods (they always get stuck on first schedule before the proxy is up)
7. Wait for Cilium to initialize

The whole thing was scriptable. Within a couple of minutes, all three nodes were Ready:

```
$ kubectl get nodes -o wide | grep -E 'manderly|norcross|oakheart|payne'
manderly    Ready    <none>   10m     v1.34.5   10.4.0.22   Ubuntu 25.10
norcross    Ready    <none>   2m47s   v1.34.5   10.4.0.23   Ubuntu 25.10
oakheart    Ready    <none>   2m24s   v1.34.5   10.4.0.24   Ubuntu 25.10
payne       Ready    <none>   118s    v1.34.5   10.4.0.25   Ubuntu 25.10
```

## The Full Bramble

Seventeen nodes, all Ready:

```
NAME        STATUS   ROLES           VERSION   OS                IP
allyrion    Ready    control-plane   v1.34.0   Talos (v1.12.5)   10.4.0.10
bettley     Ready    control-plane   v1.34.0   Talos (v1.12.5)   10.4.0.11
cargyll     Ready    control-plane   v1.34.0   Talos (v1.12.5)   10.4.0.12
dalt        Ready    <none>          v1.34.0   Talos (v1.12.5)   10.4.0.13
erenford    Ready    <none>          v1.34.0   Talos (v1.12.5)   10.4.0.14
fenn        Ready    <none>          v1.34.0   Talos (v1.12.5)   10.4.0.15
gardener    Ready    <none>          v1.34.0   Talos (v1.12.5)   10.4.0.16
harlton     Ready    <none>          v1.34.0   Talos (v1.12.5)   10.4.0.17
inchfield   Ready    <none>          v1.34.0   Talos (v1.12.5)   10.4.0.18
jast        Ready    <none>          v1.34.0   Talos (v1.12.5)   10.4.0.19
karstark    Ready    <none>          v1.34.0   Talos (v1.12.5)   10.4.0.20
lipps       Ready    <none>          v1.34.0   Talos (v1.12.5)   10.4.0.21
manderly    Ready    <none>          v1.34.5   Ubuntu 25.10      10.4.0.22
norcross    Ready    <none>          v1.34.5   Ubuntu 25.10      10.4.0.23
oakheart    Ready    <none>          v1.34.5   Ubuntu 25.10      10.4.0.24
payne       Ready    <none>          v1.34.5   Ubuntu 25.10      10.4.0.25
velaryon    Ready    <none>          v1.34.0   Talos (v1.12.5)   10.4.0.30
```

Three control plane nodes, twelve Pi 4B workers on Talos, four Pi 5 workers on Ubuntu with 932GB NVMe each, and one x86 GPU node. A mixed-OS Kubernetes cluster held together by a shared bootstrap token, a socat proxy, and a refusal to let a kernel bug win.
