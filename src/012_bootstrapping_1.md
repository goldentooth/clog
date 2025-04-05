# Bootstrapping the First Control Plane Node

With a solid idea of what it is that `kubeadm init` actually does, we can return to our command:

```
kubeadm init \
  --control-plane-endpoint="10.4.0.10:6443" \
  --kubernetes-version="stable-1.29" \
  --service-cidr="172.16.0.0/20" \
  --pod-network-cidr="192.168.0.0/16" \
  --cert-dir="/etc/kubernetes/pki" \
  --cri-socket="unix:///var/run/containerd/containerd.sock" \
  --upload-certs
```

It's really pleasantly concise, given how much is going on under the hood.

The [Ansible tasks](https://github.com/goldentooth/cluster/blob/main/roles/goldentooth.bootstrap_k8s/tasks/main.yaml) also symlinks the `/etc/kubernetes/admin.conf` file to `~/.kube/config` (so we can use `kubectl` without having to specify the config file).

Then it sets up my preferred Container Network Interface addon, Calico. I have in the past sometimes used Flannel, but Flannel doesn't support NetworkPolicy resources as it is a Layer 3 networking solution, whereas Calico operates at Layer 3 and Layer 4, which allows it fine-grained control over traffic based on ports, protocol types, sources and destinations, etc.

I want to play with NetworkPolicy resources, so Calico it is.

The next couple of steps create bootstrap tokens so we can join the cluster.
