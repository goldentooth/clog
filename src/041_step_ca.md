# Step-CA

Apparently, another thing I did recently was to set up Nomad, but I didn't take any notes about it.

That's not really that big of a deal, though, because what I _need_ to do is to get Nomad and Consul and Vault working together, and currently they aren't.

This is complicated by the fact that if I _do_ want AutoEncrypt working between Nomad and Consul, the two have to have a certificate chain proceeding from either 1) the same root certificate, or 2) different root certificates that have cross-signed. Currently, Vault has its own root certificate that I generate from scratch with the Ansible x509 tools, and then Nomad and Consul generate their own certificates using the built-in tools.

This seems messy, so it's probably time to dive into some kind of meaningful, long-term TLS infrastructure.

The choice seemed fairly clear: [`step-ca`](https://smallstep.com/docs/step-ca/). Although I hadn't used it before, I'd flirted with it a time or two and it seemed to be fairly straightforward.

I poked around a bit in [other people's implementations](https://github.com/maxhoesel-ansible/ansible-collection-smallstep/tree/main) and pilfered them ruthlessly (I've bought Max Hösel a couple coffees and I'm crediting him, never fear). I don't really need the full range of his features (and they are wonderful, it's really a lovely collection), so I cribbed the basic flow.

Once that's done, we have a few new Ansible playbooks:
- `apt_smallstep`: Configure the Smallstep Apt repository.
- `install_step_ca`: Install `step-ca` and `step-cli` on the CA node (which I've set to be Jast, the tenth node).
- `install_step_cli`: Performed on all nodes.
- `init_cluster_ca`: Initialize the certificate authority on the CA node.
- `bootstrap_cluster_ca`: Install the root certificate in the trust store on every node.
- `zap_cluster_ca`: To clean up, just nuke every file in the `step-ca` data directory.

The playbooks mentioned above get us most of the way there, but we need to revisit some of the places we've generated certificates (Vault, Consul, and Nomad) and integrate them into this system.

## Refactoring HashiApp certificate management.

As it turned out, doing that involved refactor a good amount of my Ansible IaC. One thing I've learned about code quality:

> Can you make the change easily? If so, make the change. If not, fix the most obvious obstacle, then reevaluate.

In this case, the change in question was to use the `step` CLI tool to generate certificates signed by the `step-ca` root certificate authority for services like Nomad, Vault, and Consul.

I knew immediately this would not be an easy change to make, just because of how I had written my Ansible roles. I had adopted conventional patterns for these roles, even though I knew they were not for general use and I didn't really have much intention of distributing them. Conventional patterns included naming variables expecting them to be reused across modules, etc. So I would declare variables in a general fashion within the `defaults/main.yaml` and then override them within my inventory's `group_vars` and `host_vars`.

I now consider this to be a mistake. In reality, the modules weren't _really_ designed cleanly; there were a lot of assumptions based on my own use cases that I baked into the modules, and that affected which modules I declared, etc. So yeah, I had an Ansible role to set up Slurm, but it was by no means general enough to actually help most people set up Slurm. It just gathered together a lot of tasks that I found appropriate that had to do with setting up Slurm.

Nevertheless, I persisted for a while. Mostly, I think, out of a belief that I should at least pay lip service to community style guidelines.

This task, getting Nomad and Consul and Vault working with TLS courtesy of `step-ca`, was my breaking point. There was just too much crap that needed to be renamed, just to maintain the internal consistency of an increasingly clumsy architecture intended to please people who didn't notice and almost surely wouldn't care if they had.

So, **TL;DR**: there was a great reduction in redundancy and I shifted to specifying variables in dictionaries rather than distinctly-named snake-cased variables that reminded me a little too much of Java naming conventions.

## Configuring HashiApps to use Step-CA

Once refactoring was done, configuring the apps to use Step-CA was mostly straightforward. A single `step` command was needed to generate the certificates, then another Ansible block to adjust the permissions and ownership of the generated files. For our labors, we're eventually greeted with Consul, Vault, and Nomad running exactly as they had before, but secured by a coherent certificate chain that can span all Goldentooth services.
