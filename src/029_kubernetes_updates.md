# Kubernetes Updates

Because I'm not a particularly smart man, I've allowed my cluster to fall behind. The current version, as of today, is 1.32.3, and my cluster is on 1.29.something.

So that means I need to upgrade 1.29 -> 1.30, 1.30 -> 1.31, and 1.31 -> 1.32.

## 1.29 -> 1.30

First, I update the repo URL in `/etc/apt/sources.list.d/kubernetes.sources` and run:

```bash
$ sudo apt update
Hit:1 http://deb.debian.org/debian bookworm InRelease
Hit:2 http://deb.debian.org/debian-security bookworm-security InRelease
Hit:3 http://deb.debian.org/debian bookworm-updates InRelease
Hit:4 https://download.docker.com/linux/debian bookworm InRelease
Hit:6 http://archive.raspberrypi.com/debian bookworm InRelease
Hit:7 https://baltocdn.com/helm/stable/debian all InRelease
Get:5 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb  InRelease [1,189 B]
Err:5 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb  InRelease
  The following signatures were invalid: EXPKEYSIG 234654DA9A296436 isv:kubernetes OBS Project <isv:kubernetes@build.opensuse.org>
Reading package lists... Done
W: https://download.docker.com/linux/debian/dists/bookworm/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
W: GPG error: https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb  InRelease: The following signatures were invalid: EXPKEYSIG 234654DA9A296436 isv:kubernetes OBS Project <isv:kubernetes@build.opensuse.org>
E: The repository 'https://pkgs.k8s.io/core:/stable:/v1.30/deb  InRelease' is not signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
```

Well, shit. Looks like I need to do some surgery elsewhere.

Fortunately, I had some code for setting up the Kubernetes package repositories in `install_k8s_packages`. Of course, I don't want to install new versions of the packages – the upgrade process is a little more delicate than that – so I factored it out into a new role called `setup_k8s_apt`. Running that role against my cluster with `goldentooth setup_k8s_apt` made the necessary changes.

```bash
$ sudo apt-cache madison kubeadm
   kubeadm | 1.30.11-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.10-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.9-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.8-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.7-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.6-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.5-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubeadm | 1.30.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
```

There we go. That wasn't that bad.

Now, the next steps are things I'm going to do repeatedly, and I don't want to type a bunch of commands, so I'm going to do it in Ansible. I need to do that advisedly, though.

I created a new role, `goldentooth.upgrade_k8s`. I'm working through the [upgrade documentation](https://v1-30.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/), Ansible-izing it as I go.

So I added some tasks to update the Apt cache, unhold kubeadm, upgrade it, and then hold it again (via a handler). I tagged these with `first_control_plane` and invoke the role dynamically (because that is the only context in which you can limit execution of a role to the specified tags).

```bash
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"30", GitVersion:"v1.30.11", GitCommit:"6a074997c960757de911780f250ecd9931917366", GitTreeState:"clean", BuildDate:"2025-03-11T19:56:25Z", GoVersion:"go1.23.6", Compiler:"gc", Platform:"linux/arm64"}
```

It worked!

The plan operation similarly looks fine.

```bash
$ sudo kubeadm upgrade plan
[preflight] Running pre-flight checks.
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: 1.29.6
[upgrade/versions] kubeadm version: v1.30.11
I0403 11:18:34.338987  564280 version.go:256] remote version is much newer: v1.32.3; falling back to: stable-1.30
[upgrade/versions] Target version: v1.30.11
[upgrade/versions] Latest version in the v1.29 series: v1.29.15

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   NODE        CURRENT   TARGET
kubelet     bettley     v1.29.2   v1.29.15
kubelet     cargyll     v1.29.2   v1.29.15
kubelet     dalt        v1.29.2   v1.29.15
kubelet     erenford    v1.29.2   v1.29.15
kubelet     fenn        v1.29.2   v1.29.15
kubelet     gardener    v1.29.2   v1.29.15
kubelet     harlton     v1.29.2   v1.29.15
kubelet     inchfield   v1.29.2   v1.29.15
kubelet     jast        v1.29.2   v1.29.15

Upgrade to the latest version in the v1.29 series:

COMPONENT                 NODE      CURRENT    TARGET
kube-apiserver            bettley   v1.29.6    v1.29.15
kube-apiserver            cargyll   v1.29.6    v1.29.15
kube-apiserver            dalt      v1.29.6    v1.29.15
kube-controller-manager   bettley   v1.29.6    v1.29.15
kube-controller-manager   cargyll   v1.29.6    v1.29.15
kube-controller-manager   dalt      v1.29.6    v1.29.15
kube-scheduler            bettley   v1.29.6    v1.29.15
kube-scheduler            cargyll   v1.29.6    v1.29.15
kube-scheduler            dalt      v1.29.6    v1.29.15
kube-proxy                          1.29.6     v1.29.15
CoreDNS                             v1.11.1    v1.11.3
etcd                      bettley   3.5.10-0   3.5.15-0
etcd                      cargyll   3.5.10-0   3.5.15-0
etcd                      dalt      3.5.10-0   3.5.15-0

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.29.15

_____________________________________________________________________

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   NODE        CURRENT   TARGET
kubelet     bettley     v1.29.2   v1.30.11
kubelet     cargyll     v1.29.2   v1.30.11
kubelet     dalt        v1.29.2   v1.30.11
kubelet     erenford    v1.29.2   v1.30.11
kubelet     fenn        v1.29.2   v1.30.11
kubelet     gardener    v1.29.2   v1.30.11
kubelet     harlton     v1.29.2   v1.30.11
kubelet     inchfield   v1.29.2   v1.30.11
kubelet     jast        v1.29.2   v1.30.11

Upgrade to the latest stable version:

COMPONENT                 NODE      CURRENT    TARGET
kube-apiserver            bettley   v1.29.6    v1.30.11
kube-apiserver            cargyll   v1.29.6    v1.30.11
kube-apiserver            dalt      v1.29.6    v1.30.11
kube-controller-manager   bettley   v1.29.6    v1.30.11
kube-controller-manager   cargyll   v1.29.6    v1.30.11
kube-controller-manager   dalt      v1.29.6    v1.30.11
kube-scheduler            bettley   v1.29.6    v1.30.11
kube-scheduler            cargyll   v1.29.6    v1.30.11
kube-scheduler            dalt      v1.29.6    v1.30.11
kube-proxy                          1.29.6     v1.30.11
CoreDNS                             v1.11.1    v1.11.3
etcd                      bettley   3.5.10-0   3.5.15-0
etcd                      cargyll   3.5.10-0   3.5.15-0
etcd                      dalt      3.5.10-0   3.5.15-0

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.30.11

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________
```

Of course, I won't automate the actual upgrade process; that seems unwise.

I'm skipping certificate renewal because I'd like to fight with one thing at a time.

```bash
$ sudo kubeadm upgrade apply v1.30.11 --certificate-renewal=false
[preflight] Running pre-flight checks.
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[upgrade] Running cluster health checks
[upgrade/version] You have chosen to change the cluster version to "v1.30.11"
[upgrade/versions] Cluster version: v1.29.6
[upgrade/versions] kubeadm version: v1.30.11
[upgrade] Are you sure you want to proceed? [y/N]: y
[upgrade/prepull] Pulling images required for setting up a Kubernetes cluster
[upgrade/prepull] This might take a minute or two, depending on the speed of your internet connection
[upgrade/prepull] You can also perform this action in beforehand using 'kubeadm config images pull'
W0403 11:23:42.086815  566901 checks.go:844] detected that the sandbox image "registry.k8s.io/pause:3.8" of the container runtime is inconsistent with that used by kubeadm.It is recommended to use "registry.k8s.io/pause:3.9" as the CRI sandbox image.
[upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.30.11" (timeout: 5m0s)...
[upgrade/etcd] Upgrading to TLS for etcd
[upgrade/staticpods] Preparing for "etcd" upgrade
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/etcd.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-04-03-11-25-50/etcd.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This can take up to 5m0s
[apiclient] Found 3 Pods for label selector component=etcd
[upgrade/staticpods] Component "etcd" upgraded successfully!
[upgrade/etcd] Waiting for etcd to become available
[upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests1796562509"
[upgrade/staticpods] Preparing for "kube-apiserver" upgrade
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-04-03-11-25-50/kube-apiserver.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This can take up to 5m0s
[apiclient] Found 3 Pods for label selector component=kube-apiserver
[upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-04-03-11-25-50/kube-controller-manager.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This can take up to 5m0s
[apiclient] Found 3 Pods for label selector component=kube-controller-manager
[upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-scheduler" upgrade
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-04-03-11-25-50/kube-scheduler.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This can take up to 5m0s
[apiclient] Found 3 Pods for label selector component=kube-scheduler
[upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config2173844632/config.yaml
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[upgrade/addons] skip upgrade addons because control plane instances [cargyll dalt] have not been upgraded

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.30.11". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

The next steps for the other two control plane nodes are fairly straightforward. This mostly just consisted of duplicating the playbook block to add a new step for when the playbook is executed with the 'other_control_plane' tag and adding that tag to the steps already added in the `setup_k8s` role.

```bash
$ goldentooth command cargyll,dalt 'sudo kubeadm upgrade node'
```

And a few minutes later, both of the remaining control plane nodes have updated.

The next step is to upgrade the kubelet in each node.

Serially, for obvious reasons, we need to drain each node (from a control plane node), upgrade the kubelet (unhold, upgrade, hold), then uncordon the node (again, from a control plane node).

That's not too bad, and it's included in the latest changes to the `upgrade_k8s` role.

The final step is upgrading `kubectl` on each of the control plane nodes, which is a comparative cakewalk.

```bash
$ sudo kubectl version
Client Version: v1.30.11
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
Server Version: v1.30.11
```

Nice!

## 1.30 -> 1.31

Now that the Ansible playbook and role are fleshed out, the process moving forward is comparatively simple.

1. Change the `k8s_version_clean` variable to `1.31`.
2. `goldentooth setup_k8s_apt`
3. `goldentooth upgrade_k8s --tags=kubeadm_first`
4. `goldentooth command bettley 'kubeadm version'`
5. `goldentooth command bettley 'sudo kubeadm upgrade plan'`
6. `goldentooth command bettley 'sudo kubeadm upgrade apply v1.31.7 --certificate-renewal=false -y'`
7. `goldentooth upgrade_k8s --tags=kubeadm_rest`
8. `goldentooth command cargyll,dalt 'sudo kubeadm upgrade node'`
9. `goldentooth upgrade_k8s --tags=kubelet`
10. `goldentooth upgrade_k8s --tags=kubectl`

## 1.31 -> 1.32

Hell, this is kinda fun now.

1. Change the `k8s_version_clean` variable to `1.32`.
2. `goldentooth setup_k8s_apt`
3. `goldentooth upgrade_k8s --tags=kubeadm_first`
4. `goldentooth command bettley 'kubeadm version'`
5. `goldentooth command bettley 'sudo kubeadm upgrade plan'`
6. `goldentooth command bettley 'sudo kubeadm upgrade apply v1.32.3 --certificate-renewal=false -y'`
7. `goldentooth upgrade_k8s --tags=kubeadm_rest`
8. `goldentooth command cargyll,dalt 'sudo kubeadm upgrade node'`
9. `goldentooth upgrade_k8s --tags=kubelet`
10. `goldentooth upgrade_k8s --tags=kubectl`

And eventually, everything is fine:

```bash
$ sudo kubectl get nodes
NAME        STATUS   ROLES           AGE    VERSION
bettley     Ready    control-plane   286d   v1.32.3
cargyll     Ready    control-plane   286d   v1.32.3
dalt        Ready    control-plane   286d   v1.32.3
erenford    Ready    <none>          286d   v1.32.3
fenn        Ready    <none>          286d   v1.32.3
gardener    Ready    <none>          286d   v1.32.3
harlton     Ready    <none>          286d   v1.32.3
inchfield   Ready    <none>          286d   v1.32.3
jast        Ready    <none>          286d   v1.32.3
```