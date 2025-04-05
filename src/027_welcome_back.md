# Welcome Back

So, uh, it's been a while. Things got busy and I didn't really touch the cluster for a while, and now I'm interested in it again and of course have completely forgotten everything about it.

I also ditched my OPNsense firewall because I felt it was probably costing too much power and replaced with with a simpler Unifi device, which is great but I just realized that I now have to reconfigure MetalLB to use Layer 2 instead of BGP. I probably should've used Layer 2 from the beginning, but I thought BGP would make me look a little cooler. So no load balancer integration is working right now on the cluster, which means I can't easily check in on ArgoCD. But that's fine, that's not really my highest priority.

Also, I have some new interests; I've gotten into HPC and MLOps, and some of the people I'm interested in working with use Nomad, which I've used for a couple of throwaway play projects but never on an ongoing basis. So I'm going to set up Slurm and Nomad and probably some other things. Should be fun and teach me a good amount. Of course, that's moving away from Kubernetes, but I figure I'll keep the name of this blog the same because frankly I just don't have any interest in renaming it.

First, though, I need to make sure the cluster itself is up to snuff.

Now, even I remember that I have a [little Bash tool](https://github.com/goldentooth/bash) to ease administering the cluster. And because I know me, it has online help:

```bash
$ goldentooth
Usage: goldentooth <subcommand> [arguments...]

Subcommands:
            autocomplete Enable bash autocompletion.
                 install Install Ansible dependencies.
                    lint Lint all roles.
                    ping Ping all hosts.
                  uptime Get uptime for all hosts.
                 command Run an arbitrary command on all hosts.
              edit_vault Edit the vault.
        ansible_playbook Run a specified Ansible playbook.
                   usage Display usage information.
           bootstrap_k8s Bootstrap Kubernetes cluster with kubeadm.
                 cleanup Perform various cleanup tasks.
       configure_cluster Configure the hosts in the cluster.
          install_argocd Install Argo CD on Kubernetes cluster.
     install_argocd_apps Install Argo CD applications.
            install_helm Install Helm on Kubernetes cluster.
    install_k8s_packages Install Kubernetes packages.
               reset_k8s Reset Kubernetes cluster with kubeadm.
     setup_load_balancer Setup the load balancer.
                shutdown Cleanly shut down the hosts in the cluster.
  uninstall_k8s_packages Uninstall Kubernetes packages.
```

so I can ping all of the nodes:

```bash
$ goldentooth ping
allyrion | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
gardener | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
inchfield | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
cargyll | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
erenford | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
dalt | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
bettley | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
jast | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
harlton | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
fenn | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

and... yes, that's all of them. Okay, that's a good sign.

And then I can get their uptime:

```bash
$ goldentooth uptime
gardener | CHANGED | rc=0 >>
 19:26:23 up 17 days,  4:48,  0 user,  load average: 0.13, 0.17, 0.14
allyrion | CHANGED | rc=0 >>
 19:26:23 up 17 days,  4:49,  0 user,  load average: 0.10, 0.06, 0.01
inchfield | CHANGED | rc=0 >>
 19:26:23 up 17 days,  4:47,  0 user,  load average: 0.25, 0.59, 0.60
erenford | CHANGED | rc=0 >>
 19:26:23 up 17 days,  4:48,  0 user,  load average: 0.08, 0.15, 0.12
jast | CHANGED | rc=0 >>
 19:26:23 up 17 days,  4:47,  0 user,  load average: 0.11, 0.19, 0.27
dalt | CHANGED | rc=0 >>
 19:26:23 up 17 days,  4:49,  0 user,  load average: 0.84, 0.64, 0.59
fenn | CHANGED | rc=0 >>
 19:26:23 up 17 days,  4:48,  0 user,  load average: 0.27, 0.34, 0.23
harlton | CHANGED | rc=0 >>
 19:26:23 up 17 days,  4:48,  0 user,  load average: 0.27, 0.14, 0.20
bettley | CHANGED | rc=0 >>
 19:26:23 up 17 days,  4:49,  0 user,  load average: 0.41, 0.49, 0.81
cargyll | CHANGED | rc=0 >>
 19:26:23 up 17 days,  4:49,  0 user,  load average: 0.26, 0.42, 0.64
```

17 days, which is about when I set up the new router and had to reorganize a lot of my network. Seems legit. So it looks like the power supplies are still fine. When I first set up the cluster, I think there was a flaky USB cable on one of the Pis, so it would occasionally drop off. I'd prefer to control my chaos engineering, not have it arise spontaneously from my poor QC, thank you very much.

My first node just runs HAProxy (currently) and is the simplest, so I'm going to check and see what needs to be updated. Nobody cares about `apt` stuff so I'll skip the details.

**TL;DR**: It wasn't that much, really, though it does appear that I had some files in `/etc/modprobe.d` that should've been in `/etc/modules-load.d`. I blame... someone else.

So I'll update all of the nodes, hope they rejoin the cluster when they reboot, and in the next entry I'll try to update Kubernetes...
