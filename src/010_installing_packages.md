# Installing Packages

Now that we have functional access to the Kubernetes Apt package repository, we can install some important Kubernetes tools:

- `kubeadm` provides a straightforward way to setup and configure a Kubernetes cluster (API server, Controller Manager, DNS, etc). _Kubernetes the Hard Way_ basically does what `kubeadm` does. I use `kubeadm` because my goal is to go not necessarily _deeper_, but _farther_.
- `kubectl` is a CLI tool for administering a Kubernetes cluster; you can deploy applications, inspect resources, view logs, etc. As I'm studying for my CKA, I want to use `kubectl` for as much as possible.
- `kubelet` runs on each and every node in the cluster and ensures that pods are functioning as desired and takes steps to correct their behavior when it does not match the desired state.

Installing these tools is comparatively simple, just `sudo apt-get install -y kubeadm kubectl kubelet`, or as covered in [the relevant role](https://github.com/goldentooth/cluster/blob/main/roles/goldentooth.install_k8s_packages/tasks/main.yaml).
