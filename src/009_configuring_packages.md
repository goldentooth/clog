# Configuring Packages

Rather than YOLOing binaries onto our nodes like heathens, we'll use Apt and Ansible.

I wrote the above line before a few hours or so of fighting with Apt, Ansible, the repository signing key, documentation on the greater internet, my emotions, etc.

The long and short of it is that `apt-key add` is deprecated in Debian and Ubuntu, and consequently `ansible.builtin.apt_key` _should_ be deprecated, but cannot be at this time for backward compatibility with older versions of Debian and Ubuntu and other derivative distributions.

The reason for this deprecation, as I understand it, is that `apt-key add` adds a key to `/etc/apt/trusted.gpg.d`. Keys here can be used to sign any package, including a package downloaded from an official distro package repository. This weakens our defenses against supply-chain attacks.

The new recommendation is to add the key to `/etc/apt/keyrings`, where it will be used when appropriate but not, apparently, to sign for official distro package repositories.

A further complication is that the Kubernetes project has moved its package repositories a time or two and completely rewrote the repository structure.

As a result, if you Google™, you will find a number of ways of using Ansible or a shell command to configure the Kubernetes apt repository on Debian/Ubuntu/Raspberry Pi OS, but none of them are optimal.

## The Desired End-State

Here are my expectations:

- use the new deb822 format, not the old sources.list format
- preserve idempotence
- don't point to deprecated package repositories
- actually work

Existing solutions failed at one or all of these.

For the record, what we're trying to create is:
- a file located at `/etc/apt/keyrings/kubernetes.asc` containing the Kubernetes package repository signing key
- a file located at `/etc/apt/sources.list.d/kubernetes.sources` containing information about the Kubernetes package repository.

The latter should look something like the following:

```
X-Repolib-Name: kubernetes
Types: deb
URIs: https://pkgs.k8s.io/core:/stable:/v1.29/deb/
Suites: /
Architectures: arm64
Signed-By: /etc/apt/keyrings/kubernetes.asc
```

## The Solution

After quite some time and effort and suffering, I arrived at a solution.

You can review the [original task file](https://github.com/goldentooth/cluster/blob/main/roles/goldentooth.install_k8s_packages/tasks/main.yaml) for changes, but I'm embedding it here because it was weirdly a nightmare to arrive at a working solution.

I've edited this only to substitute strings for the variables that point to them, so it should be a working solution more-or-less out-of-the-box.

```ansible
---
- name: 'Install packages needed to use the Kubernetes Apt repository.'
  ansible.builtin.apt:
    name:
      - 'apt-transport-https'
      - 'ca-certificates'
      - 'curl'
      - 'gnupg'
      - 'python3-debian'
    state: 'present'

- name: 'Add Kubernetes repository.'
  ansible.builtin.deb822_repository:
    name: 'kubernetes'
    types:
      - 'deb'
    uris:
      - "https://pkgs.k8s.io/core:/stable:/v1.29/deb/"
    suites:
      - '/'
    architectures:
      - 'arm64'
    signed_by: "https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key"
```

After this, you will of course need to update your Apt cache and install the three Kubernetes tools we'll use shortly: `kubeadm`, `kubectl`, and `kubelet`.
