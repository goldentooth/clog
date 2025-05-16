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

This gets us most of the way there, but we need to revisit some of the places we've generated certificates (Vault, Consul, and Nomad) and integrate them into this system.

