## Consul

I wanted to install a service discovery system to manage, well, all of the other services that exist only to manage other services on this cluster.

I have the idea of installing Authelia, then Envoy, then Consul in a chain as a replacement for Nginx. Obviously it's far more complicated than Nginx, but by now that's the point; to increase the complexity of this homelab until it collapses under its own weight. Alas poor GoldenTooth. I knew him, Gentle Reader, a cluster of infinite GPIO!

First order of business is to set up the Consul servers – leader and followers – which will occupy Bettley, Cargyll, and Dalt.

For most of this, I just followed the [deployment guide](https://developer.hashicorp.com/consul/tutorials/production-vms/deployment-guide#install-consul). Then I followed [the guide for creating client agent tokens](https://developer.hashicorp.com/consul/docs/security/acl/tokens/create/create-an-agent-token).

Unfortunately, I encountered some issues when it came to setting up ACLs. For some reason, my server nodes worked precisely as expected, but my nodes would not join the cluster.

```log
Apr 12 13:44:56 fenn consul[328873]: ==> Starting Consul agent...
Apr 12 13:44:56 fenn consul[328873]:                Version: '1.20.5'
Apr 12 13:44:56 fenn consul[328873]:             Build Date: '2025-03-11 10:16:18 +0000 UTC'
Apr 12 13:44:56 fenn consul[328873]:                Node ID: 'a5c6a1f2-8811-9de7-917f-acc1cd9fc8b7'
Apr 12 13:44:56 fenn consul[328873]:              Node name: 'fenn'
Apr 12 13:44:56 fenn consul[328873]:             Datacenter: 'dc1' (Segment: '')
Apr 12 13:44:56 fenn consul[328873]:                 Server: false (Bootstrap: false)
Apr 12 13:44:56 fenn consul[328873]:            Client Addr: [127.0.0.1] (HTTP: 8500, HTTPS: -1, gRPC: -1, gRPC-TLS: -1, DNS: 8600)
Apr 12 13:44:56 fenn consul[328873]:           Cluster Addr: 10.4.0.15 (LAN: 8301, WAN: 8302)
Apr 12 13:44:56 fenn consul[328873]:      Gossip Encryption: true
Apr 12 13:44:56 fenn consul[328873]:       Auto-Encrypt-TLS: true
Apr 12 13:44:56 fenn consul[328873]:            ACL Enabled: true
Apr 12 13:44:56 fenn consul[328873]:     ACL Default Policy: deny
Apr 12 13:44:56 fenn consul[328873]:              HTTPS TLS: Verify Incoming: true, Verify Outgoing: true, Min Version: TLSv1_2
Apr 12 13:44:56 fenn consul[328873]:               gRPC TLS: Verify Incoming: true, Min Version: TLSv1_2
Apr 12 13:44:56 fenn consul[328873]:       Internal RPC TLS: Verify Incoming: true, Verify Outgoing: true (Verify Hostname: true), Min Version: TLSv1_2
Apr 12 13:44:56 fenn consul[328873]: ==> Log data will now stream in as it occurs:
Apr 12 13:44:56 fenn consul[328873]: 2025-04-12T13:44:55.999-0400 [WARN]  agent: skipping file /etc/consul.d/consul.env, extension must be .hcl or .json, or config f
ormat must be set
Apr 12 13:44:56 fenn consul[328873]: 2025-04-12T13:44:56.021-0400 [WARN]  agent.auto_config: skipping file /etc/consul.d/consul.env, extension must be .hcl or .json,
 or config format must be set
Apr 12 13:45:06 fenn consul[328873]: 2025-04-12T13:45:06.216-0400 [ERROR] agent.auto_config: AutoEncrypt.Sign RPC failed: addr=10.4.0.11:8300 error="rpcinsecure: err
or making call: rpcinsecure: error making call: Permission denied: anonymous token lacks permission 'node:write' on \"fenn\". The anonymous token is used implicitly
when a request does not specify a token."
Apr 12 13:45:06 fenn consul[328873]: 2025-04-12T13:45:06.225-0400 [ERROR] agent.auto_config: AutoEncrypt.Sign RPC failed: addr=10.4.0.12:8300 error="rpcinsecure: err
or making call: Permission denied: anonymous token lacks permission 'node:write' on \"fenn\". The anonymous token is used implicitly when a request does not specify
a token."
Apr 12 13:45:06 fenn consul[328873]: 2025-04-12T13:45:06.240-0400 [ERROR] agent.auto_config: AutoEncrypt.Sign RPC failed: addr=10.4.0.13:8300 error="rpcinsecure: error making call: rpcinsecure: error making call: Permission denied: anonymous token lacks permission 'node:write' on \"fenn\". The anonymous token is used implicitly when a request does not specify a token."
Apr 12 13:45:06 fenn consul[328873]: 2025-04-12T13:45:06.240-0400 [ERROR] agent.auto_config: AutoEncrypt.Sign RPC failed: addr=10.4.0.13:8300 error="rpcinsecure: error making call: rpcinsecure: error making call: Permission denied: anonymous token lacks permission 'node:write' on \"fenn\". The anonymous token is used implicitly when a request does not specify a token."
Apr 12 13:45:06 fenn consul[328873]: 2025-04-12T13:45:06.240-0400 [ERROR] agent.auto_config: No servers successfully responded to the auto-encrypt request
Apr 12 13:45:06 fenn consul[328873]: 2025-04-12T13:45:06.255-0400 [ERROR] agent.auto_config: AutoEncrypt.Sign RPC failed: addr=10.4.0.11:8300 error="rpcinsecure: error making call: rpcinsecure: error making call: Permission denied: anonymous token lacks permission 'node:write' on \"fenn\". The anonymous token is used implicitly when a request does not specify a token."
Apr 12 13:45:06 fenn consul[328873]: 2025-04-12T13:45:06.263-0400 [ERROR] agent.auto_config: AutoEncrypt.Sign RPC failed: addr=10.4.0.12:8300 error="rpcinsecure: error making call: Permission denied: anonymous token lacks permission 'node:write' on \"fenn\". The anonymous token is used implicitly when a request does not specify a token."
Apr 12 13:45:06 fenn consul[328873]: 2025-04-12T13:45:06.277-0400 [ERROR] agent.auto_config: AutoEncrypt.Sign RPC failed: addr=10.4.0.13:8300 error="rpcinsecure: error making call: rpcinsecure: error making call: Permission denied: anonymous token lacks permission 'node:write' on \"fenn\". The anonymous token is used implicitly when a request does not specify a token."
Apr 12 13:45:06 fenn consul[328873]: 2025-04-12T13:45:06.277-0400 [ERROR] agent.auto_config: No servers successfully responded to the auto-encrypt request
Apr 12 13:45:07 fenn consul[328873]: 2025-04-12T13:45:07.508-0400 [ERROR] agent.auto_config: AutoEncrypt.Sign RPC failed: addr=10.4.0.11:8300 error="rpcinsecure: error making call: rpcinsecure: error making call: Permission denied: anonymous token lacks permission 'node:write' on \"fenn\". The anonymous token is used implicitly when a request does not specify a token."
Apr 12 13:45:07 fenn consul[328873]: 2025-04-12T13:45:07.515-0400 [ERROR] agent.auto_config: AutoEncrypt.Sign RPC failed: addr=10.4.0.12:8300 error="rpcinsecure: error making call: Permission denied: anonymous token lacks permission 'node:write' on \"fenn\". The anonymous token is used implicitly when a request does not specify a token."
Apr 12 13:45:07 fenn consul[328873]: 2025-04-12T13:45:07.538-0400 [ERROR] agent.auto_config: AutoEncrypt.Sign RPC failed: addr=10.4.0.13:8300 error="rpcinsecure: error making call: rpcinsecure: error making call: Permission denied: anonymous token lacks permission 'node:write' on \"fenn\". The anonymous token is used implicitly when a request does not specify a token."
Apr 12 13:45:07 fenn consul[328873]: 2025-04-12T13:45:07.538-0400 [ERROR] agent.auto_config: No servers successfully responded to the auto-encrypt request
```

It seemed that the token would not be persisted on the client node after running `consul acl set-agent-token agent <acl-token-secret-id>`, even though I have `enable_token_persistence` set to `true`. As a result, I needed to go back and set it in the `consul.hcl` configuration file.

The fiddliness of the ACL bootstrapping also led me to split that out into a separate Ansible role.
