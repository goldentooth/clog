# Talos

At this point, I was quite a ways into this project; about 15-18 months of off-and-on work. It was very complex, with a long list of Ansible playbooks and roles, a custom CLI, several additional repositories for GitOps, a half-implemented AI agent and a half-implemented MCP server, multiple filesystems, complex mesh networking...

The complexity was (and still is) a large part of the point; I wanted a bustling, lively cluster full of ephemeral, dispensable services, with an architecture in the fashion of one designed by multiple teams organized chaotically. A lot of chatter, a lot of noise. Much ado about nothing. That's all fine, and that was working more-or-less as intended.

But this month (September 2025) I found myself dissatisfied and yearning for the Holy Grail of infrastructure, which (currently at least) can be summarized as everything being declarative.

Back when I first tried Kubernetes, in... IDK, 2016 or 2017... I quickly became frustrated with running Kubernetes on Debian and made the leap to CoreOS, which is confusingly different from what I now understand to be CoreOS and is closer to what I believe is called Flatcar Linux. Apologies if I'm getting any of that wrong; the reasons why will likely beome clear.

I had four old Dell Optiplex PCs, which I threw 32GB RAM apiece on, installed VMWare (this was before I knew about Proxmox), and created four VMs on each PC so that I had a 2D matrix of 16 VMs. I set up another VM to host Matchbox and then wrote Ignition configurations (again, I might be wrong about the naming here) so that each VM, with its distinct MAC address, would PXE-boot CoreOS and form a Kubernetes cluster, then install some other manifests and host some services. I learned a lot, aged quickly, etc. I upgraded Kubernetes, I followed Ignition through its godawful rebranding as "CoreOS Matchbox Configurator" or something equally appalling, etc.

But this was a homelab, and as has generally been the pattern with my homelabs, I installed services like Plex and such that "mattered" and that needed to be "stable". So however much the VMs started out as "cattle," the cluster itself became a sort of "pet". And the complexity of the infrastructure and my lack of effective ops knowledge (compounded exponentially by my ginger treatment of the cluster) of course led to something that I didn't touch and ended up replacing with a normal "pet" server that was easier to reason about. The cluster languished and at some point I broke it down and sold the parts.

But that cluster of VMs remained a fond memory, especially because I could absolutely bork a node beyond all recognition and then... just...reboot. The specification of the cluster either worked or it didn't, and the cluster would either conform to its specification or it wouldn't; no confusion or misconfiguration arising from past states, no filesystem clutter, no need to consult my notes when an SD card shat blood or a new node was added.

So I definitely took note of Talos Linux when I first started hearing buzz about it a couple years back. But the time wasn't yet ripe. I wanted to play with some other systems - Slurm, Nomad, Docker Swarm, etc.

Now I've done that, and I think I'm fairly satisfied with my flirtations. I was very interested in Slurm and HPC, and proud of the cluster I'd built out, but I wasn't able to get any traction applying for the few relevant jobs I found in the area. Nomad's cool, but it seemed like any place that ran Nomad was equally fine with someone who had Kubernetes experience. I don't know if I ever saw anyone explicitly mention Docker Swarm.

So the few concerns I had about shifting to Talos and "fulltime" Kubernetes evaporated, and the hybrid approach of running bare-metal services and Kubernetes services and Nomad services was starting to be a PITA.

This chapter of the [CLOG](https://clog.goldentooth.net/) is therefore kind of a tombstone on the old structure and marks the cluster's transition to a new infrastructure based on Talos. Currently, I'm not netbooting (that'll come later), and not all of the nodes are running Talos (it's not supported on Pi 5s, and I haven't gotten around to installing it on Velaryon), but I have gotten it installed on the 12 RPi Bs, and I'm figuring out how to manage the Talos configuration in a GitOps-y way with Talhelper. My findings there are not terribly interesting, and given that I'm a rank newbie I'm concerned about spreading misinformation and antipatterns.

But I'll pick up in the next chapter with... something of interest. I hope.
