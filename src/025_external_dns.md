# ExternalDNS

The workflow for accessing our `LoadBalancer` services ain't great.

If we deploy a new application, we need to run `kubectl -n <namespace> get svc` and read through a list to determine the IP address on which it's exposed. And that's not going to be stable; there's nothing at all guaranteeing that Argo CD will always be available at `http://10.4.11.1`.

Enter [ExternalDNS](https://github.com/kubernetes-sigs/external-dns). The idea is that we annotate our services with `external-dns.alpha.kubernetes.io/hostname: "argocd.my-cluster.my-domain.com"` and a DNS record will be created pointing to the actual IP address of the `LoadBalancer` service.

This is comparatively straightforward to configure if you host your DNS in one of the supported services. I host mine via AWS Route53, which is supported.

The complication is that we don't yet have a great way of managing secrets, so there's a manual step here that I find unpleasant, but we'll cross that bridge when we get to it.

Because of work we've done previously with Argo CD, we can just create [a new repository](https://github.com/goldentooth/external-dns) to deploy ExternalDNS within our cluster.

This has the following manifests:

- **Deployment**: The deployment has several interesting features:
  - This is where the `--provider` (`aws`) is configured.
  - We specify `--source`s (in our case `service`).
  - A `--domain-filter` allows us to use different configurations for different domain names.
  - A `--txt-owner-id` allows us to map from a record back to the application that created it.
  - Mounts a secret  as AWS credentials (I used static credentials for the time being) so ExternalDNS can make the changes in Route53.
- **ServiceAccount**: Just adds a service account for ExternalDNS.
- **ClusterRole**: Describes an ability to observe changes in services.
- **ClusterRoleBinding**: Binds the above cluster role and ExternalDNS.

A few minutes after pushing changes to the repository, we can reach Argo CD via https://argocd.goldentooth.hellholt.net/.
