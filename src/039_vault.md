## Vault

As long as I'm setting up Consul, I figure I might as well set up Vault too.

This wasn't that bad, compared to the experience I had with ACLs in Consul. I set up a KMS key for unsealing, generated a certificate authority and regenerated TLS assets for my three server nodes, and Raft kinda just worked.

