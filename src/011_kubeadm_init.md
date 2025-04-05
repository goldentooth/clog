# `kubeadm init`

`kubeadm` does a wonderful job of simplifying Kubernetes cluster bootstrapping (if you don't believe me, just read _Kubernetes the Hard Way_), but there's still a decent amount of work involved. Since we're creating a high-availability cluster, we need to do some magic to convey secrets between the control plane nodes, generate join tokens for the worker nodes, etc.

So, we will:
- run `kubeadm` on the first control plane node
- copy some data around
- run a different `kubeadm` command to join the rest of the control plane nodes to the cluster
- copy some more data around
- run a different `kubeadm` command to join the worker nodes to the cluster

and then we're done!

`kubeadm init` takes a number of command-line arguments.

You can look at the [actual Ansible tasks bootstrapping my cluster](https://github.com/goldentooth/cluster/blob/main/roles/goldentooth.bootstrap_k8s/tasks/main.yaml), but this is what my command evaluates out to:

```
kubeadm init \
  --control-plane-endpoint="10.4.0.10:6443" \
  --kubernetes-version="stable-1.29" \
  --service-cidr="172.16.0.0/20" \
  --pod-network-cidr="192.168.0.0/16" \
  --cert-dir="/etc/kubernetes/pki" \
  --cri-socket="unix:///var/run/containerd/containerd.sock" \
  --upload-certs
```

I'll break that down line by line:

```
# Run through all of the phases of initializing a Kubernetes control plane.
kubeadm init \
  # Requests should target the load balancer, not this particular node.
  --control-plane-endpoint="10.4.0.10:6443" \
  # We don't need any more instability than we already have.
  # At time of writing, 1.29 is the current release.
  --kubernetes-version="stable-1.29" \
  # As described in the chapter on Networking, this is the CIDR from which
  # service IP addresses will be allocated.
  # This gives us 4,094 IP addresses to work with.
  --service-cidr="172.16.0.0/20" \
  # As described in the chapter on Networking, this is the CIDR from which
  # pod IP addresses will be allocated.
  # This gives us 65,534 IP addresses to work with.
  --pod-network-cidr="192.168.0.0/16"
  # This is the directory that will host TLS certificates, keys, etc for
  # cluster communication.
  --cert-dir="/etc/kubernetes/pki"
  # This is the URI of the container runtime interface socket, which allows
  # direct interaction with the container runtime.
  --cri-socket="unix:///var/run/containerd/containerd.sock"
  # Upload certificates into the appropriate secrets, rather than making us
  # do that manually.
  --upload-certs
```

Oh, you thought I was just going to blow right by this, didncha? No, this ain't _Kubernetes the Hard Way_, but I do want to make an effort to understand what's going on here. So here, courtesy of `kubeadm init --help`, is the list of phases that `kubeadm` runs through by default.

```
preflight                    Run pre-flight checks
certs                        Certificate generation
  /ca                          Generate the self-signed Kubernetes CA to provision identities for other Kubernetes components
  /apiserver                   Generate the certificate for serving the Kubernetes API
  /apiserver-kubelet-client    Generate the certificate for the API server to connect to kubelet
  /front-proxy-ca              Generate the self-signed CA to provision identities for front proxy
  /front-proxy-client          Generate the certificate for the front proxy client
  /etcd-ca                     Generate the self-signed CA to provision identities for etcd
  /etcd-server                 Generate the certificate for serving etcd
  /etcd-peer                   Generate the certificate for etcd nodes to communicate with each other
  /etcd-healthcheck-client     Generate the certificate for liveness probes to healthcheck etcd
  /apiserver-etcd-client       Generate the certificate the apiserver uses to access etcd
  /sa                          Generate a private key for signing service account tokens along with its public key
kubeconfig                   Generate all kubeconfig files necessary to establish the control plane and the admin kubeconfig file
  /admin                       Generate a kubeconfig file for the admin to use and for kubeadm itself
  /super-admin                 Generate a kubeconfig file for the super-admin
  /kubelet                     Generate a kubeconfig file for the kubelet to use *only* for cluster bootstrapping purposes
  /controller-manager          Generate a kubeconfig file for the controller manager to use
  /scheduler                   Generate a kubeconfig file for the scheduler to use
etcd                         Generate static Pod manifest file for local etcd
  /local                       Generate the static Pod manifest file for a local, single-node local etcd instance
control-plane                Generate all static Pod manifest files necessary to establish the control plane
  /apiserver                   Generates the kube-apiserver static Pod manifest
  /controller-manager          Generates the kube-controller-manager static Pod manifest
  /scheduler                   Generates the kube-scheduler static Pod manifest
kubelet-start                Write kubelet settings and (re)start the kubelet
upload-config                Upload the kubeadm and kubelet configuration to a ConfigMap
  /kubeadm                     Upload the kubeadm ClusterConfiguration to a ConfigMap
  /kubelet                     Upload the kubelet component config to a ConfigMap
upload-certs                 Upload certificates to kubeadm-certs
mark-control-plane           Mark a node as a control-plane
bootstrap-token              Generates bootstrap tokens used to join a node to a cluster
kubelet-finalize             Updates settings relevant to the kubelet after TLS bootstrap
  /experimental-cert-rotation  Enable kubelet client certificate rotation
addon                        Install required addons for passing conformance tests
  /coredns                     Install the CoreDNS addon to a Kubernetes cluster
  /kube-proxy                  Install the kube-proxy addon to a Kubernetes cluster
show-join-command            Show the join command for control-plane and worker node
```

So now I will go through each of these in turn to explain how the cluster is created.

## `kubeadm` init phases

### `preflight`

The preflight phase performs a number of checks of the environment to ensure it is suitable. These aren't, as far as I can tell, documented anywhere -- perhaps because documentation would inevitably drift out of sync with the code rather quickly. And, besides, we're engineers and this is an open-source project; if we care that much, we can just read the [source code](https://github.com/kubernetes/kubernetes/blob/master/cmd/kubeadm/app/preflight/checks.go)!

But I'll go through and mention a few of these checks, just for the sake of discussion and because there are some important concepts.

- **Networking**: It checks that certain ports are available and firewall settings do not prevent communication.
- **Container Runtime**: It requires a container runtime, since... Kubernetes is a container orchestration platform.
- **Swap**: Historically, Kubernetes has balked at running on a system with swap enabled, for performance and stability reasons, but this has been lifted recently.
- **Uniqueness**: It checks that each hostname is different in order to prevent networking conflicts.
- **Kernel Parameters**: It checks for certain cgroups (see the Node configuration chapter for more information). It used to check for some networking parameters as well, to ensure traffic can flow properly, but it appears [this might not be a thing anymore](https://github.com/kubernetes/kubernetes/commit/75238e592d624ad57cdf700c1bb42a8e5366bcb2) in 1.30.

### `certs`

This phase generates important certificates for communication between cluster components.

#### `/ca`

This generates a self-signed certificate authority that will be used to provision identities for all of the other Kubernetes components, and lays the groundwork for the security and reliability of their communication by ensuring that all components are able to trust one another.

By generating its own root CA, a Kubernetes cluster can be self-sufficient in managing the lifecycle of the certificates it uses for TLS. This includes generating, distributing, rotating, and revoking certificates as needed. This autonomy simplifies the setup and ongoing management of the cluster, especially in environments where integrating with an external CA might be challenging.

It's worth mentioning that this includes _client_ certificates as well as server certificates, since client certificates aren't currently as well-known and ubiquitous as server certificates. So just as the API server has a server certificate that allows clients making requests to verify its identity, so clients will have a client certificate that allows the server to verify their identity.

So these certificate relationships maintain CIA (Confidentiality, Integrity, and Authentication) by:
- encrypting the data transmitted between the client and the server (Confidentiality)
- preventing tampering with the data transmitted between the client and the server (Integrity)
- verifying the identity of the server _and_ the client (Authentication)

#### `/apiserver`

The Kubernetes API server is the central management entity of the cluster. The Kubernetes API allows users and internal and external processes and components to communicate and report and manage the state of the cluster. The API server accepts, validates, and executes REST operations, and is the only cluster component that interacts with `etcd` directly. `etcd` is the source of truth within the cluster, so it is essential that communication with the API server be secure.

#### `/apiserver-kubelet-client`

This is a _client_ certificate for the API server, ensuring that it can authenticate itself to each kubelet and prove that it is a legitimate source of commands and requests.

#### `/front-proxy-ca` and `front-proxy-client`

The Front Proxy certificates seem to only be used in situations where `kube-proxy` is supporting an extension API server, and the API server/aggregator needs to connect to an extension API server respectively. This is beyond the scope of this project.

#### `/etcd-ca`

`etcd` can be configured to run "stacked" (deployed onto the control plane) or as an external cluster. For various reasons (security via isolation, access control, simplified rotation and management, etc), `etcd` is provided its own certificate authority.

#### `/etcd-server`

This is a server certificate for each `etcd` node, assuring the Kubernetes API server and `etcd` peers of its identity.

#### `/etcd-peer`

This is a client _and_ server certificate, distributed to each `etcd` node, that enables them to communicate securely with one another.

#### `/etcd-healthcheck-client`

This is a client certificate that enables the caller to probe `etcd`. It permits broader access, in that multiple clients can use it, but the degree of that access is very restricted.

#### `/apiserver-etcd-client`

This is a client certificate permitting the API server to communicate with `etcd`.

#### `/sa`

This is a public and private key pair that is used for signing service account tokens.

Service accounts are used to provide an identity for processes that run in a Pod, permitting them to interact securely with the API server.

Service account tokens are JWTs (JSON Web Tokens). When a Pod accesses the Kubernetes API, it can present a service account token as a bearer token in the HTTP Authorization header. The API server then uses the public key to verify the signature on the token, and can then evaluate whether the claims are valid, etc.

### `kubeconfig`

These phases write the necessary configuration files to secure and facilitate communication within the cluster and between administrator tools (like `kubectl`) and the cluster.

#### `/admin`

This is the kubeconfig file for the cluster administrator. It provides the admin user with full access to the cluster.

Now, per a change in 1.29, as [Rory McCune explains](https://raesene.github.io/blog/2024/01/06/when-is-admin-not-admin/), this `admin` credential is no longer a member of `system:masters` and instead has access granted via RBAC. This means that access can be revoked without having to manually rotate all of the cluster certificates.

#### `/super-admin`

This new credential also provides full access to the cluster, but via the `system:masters` group mechanism (read: irrevocable without rotating certificates). This also explains why, when watching my cluster spin up while using the `admin.conf` credentials, a time or two I saw access denied errors!

#### `/kubelet`

This credential is for use with the kubelet during cluster bootstrapping. It provides a baseline cluster-wide configuration for all kubelets in the cluster. It points to the client certificates that allow the kubelet to communicate with the API server so we can propagate cluster-level configuration to each kubelet.

#### `/controller-manager`

This credential is used by the Controller Manager. The Controller Manager is responsible for running controller processes, which watch the state of the cluster through the API server and make changes attempting to move the current state towards the desired state. This file contains credentials that allow the Controller Manager to communicate securely with the API server.

#### `/scheduler`

This credential is used by the Kubernetes Scheduler. The Scheduler is responsible for assigning work, in the form of Pods, to different nodes in the cluster. It makes these decisions based on resource availability, workload requirements, and other policies. This file contains the credentials needed for the Scheduler to interact with the API server.

### `etcd`

This phase generates the static pod manifest file for local `etcd`.

Static pod manifests are files kept in (in our case) `/etc/kubernetes/manifests`; the kubelet observes this directory and will start/replace/delete pods accordingly. In the case of a "stacked" cluster, where we have critical control plane components like `etcd` and the API server running within pods, we need some method of creating and managing pods without those components. Static pod manifests provide this capability.

#### `/local`

This phase configures a local `etcd` instance to run on the same node as the other control plane components. This is what we'll be doing; later, when we join additional nodes to the control plane, the `etcd` cluster will expand.

For instance, the static pod manifest file for `etcd` on `bettley`, my first control plane node, has a `spec.containers[0].command` that looks like this:

```
....
  - command:
    - etcd
    - --advertise-client-urls=https://10.4.0.11:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --experimental-initial-corrupt-check=true
    - --experimental-watch-progress-notify-interval=5s
    - --initial-advertise-peer-urls=https://10.4.0.11:2380
    - --initial-cluster=bettley=https://10.4.0.11:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://10.4.0.11:2379
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://10.4.0.11:2380
    - --name=bettley
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
....
```

whereas on `fenn`, the second control plane node, the corresponding static pod manifest file looks like this:

```
  - command:
    - etcd
    - --advertise-client-urls=https://10.4.0.15:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --experimental-initial-corrupt-check=true
    - --experimental-watch-progress-notify-interval=5s
    - --initial-advertise-peer-urls=https://10.4.0.15:2380
    - --initial-cluster=fenn=https://10.4.0.15:2380,gardener=https://10.4.0.16:2380,bettley=https://10.4.0.11:2380
    - --initial-cluster-state=existing
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://10.4.0.15:2379
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://10.4.0.15:2380
    - --name=fenn
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

and correspondingly, we can see three pods:

```
$ kubectl -n kube-system get pods
NAME                                       READY   STATUS    RESTARTS   AGE
etcd-bettley                               1/1     Running   19         3h23m
etcd-fenn                                  1/1     Running   0          3h22m
etcd-gardener                              1/1     Running   0          3h23m
```

### `control-plane`

This phase generates the static pod manifest files for the other (non-`etcd`) control plane components.

#### `/apiserver`

This generates the static pod manifest file for the API server, which we've already discussed quite a bit.

#### `/controller-manager`

This generates the static pod manifest file for the controller manager. The controller manager embeds the core control loops shipped with Kubernetes. A controller is a loop that watches the shared state of the cluster through the API server and makes changes attempting to move the current state towards the desired state. Examples of controllers that are part of the Controller Manager include the Replication Controller, Endpoints Controller, Namespace Controller, and ServiceAccounts Controller.

#### `/scheduler`

This phase generates the static pod manifest file for the scheduler. The scheduler is responsible for allocating pods to nodes in the cluster based on various scheduling principles, including resource availability, constraints, affinities, etc.

### `kubelet-start`

Throughout this process, the kubelet has been in a crash loop because it hasn't had a valid configuration.

This phase generates a config which (at least on my system) is stored at `/var/lib/kubelet/config.yaml`, as well as a "bootstrap" configuration that allows the kubelet to connect to the control plane (and retrieve credentials for longterm use).

Then the kubelet is restarted and will bootstrap with the control plane.

### `upload-certs`

This phase enables the secure distribution of the certificates we created above, in the `certs` phases.

Some certificates need to be shared across the cluster (or at least across the control plane) for secure communication. This includes the certificates for the API server, `etcd`, the front proxy, etc.

`kubeadm` generates an encryption key that is used to encrypt the certificates, so they're not actually exposed in plain text at any point. Then the encrypted certificates are uploaded to `etcd`, a distributed key-value store that Kubernetes uses for persisting cluster state. To facilitate future joins of control plane nodes without having to manually distribute certificates, these encrypted certificates are stored in a specific `kubeadm-certs` secret.

The encryption key is required to decrypt the certificates for use by joining nodes. This key is not uploaded to the cluster for security reasons. Instead, it must be manually shared with any future control plane nodes that join the cluster. kubeadm outputs this key upon completion of the `upload-certs` phase, and it's the administrator's responsibility to securely transfer this key when adding new control plane nodes.

This process allows for the secure addition of new control plane nodes to the cluster by ensuring they have access to the necessary certificates to communicate securely with the rest of the cluster. Without this phase, administrators would have to manually copy certificates to each new node, which can be error-prone and insecure.

By automating the distribution of these certificates and utilizing encryption for their transfer, `kubeadm` significantly simplifies the process of scaling the cluster's control plane, while maintaining high standards of security.

### `mark-control-plane`

In this phase, `kubeadm` applies a specific label to control plane nodes: `node-role.kubernetes.io/control-plane=""`; this marks the node as part of the control plane. Additionally, the node receives a taint, `node-role.kubernetes.io/control-plane=:NoSchedule`, which will prevent normal workloads from being scheduled on it.

As noted previously, I see no reason to remove this taint, although I'll probably enable some tolerations for certain workloads (monitoring, etc).

### `bootstrap-token`

This phase creates bootstrap tokens, which are used to authenticate new nodes joining the cluster. This is how we are able to easily scale the cluster dynamically without copying multiple certificates and private keys around.

The "TLS bootstrap" process allows a kubelet to automatically request a certificate from the Kubernetes API server. This certificate is then used for secure communication within the cluster. The process involves the use of a bootstrap token and a Certificate Signing Request (CSR) that the kubelet generates. Once approved, the kubelet receives a certificate and key that it uses for authenticated communication with the API server.

Bootstrap tokens are a simple bearer token. This token is composed of two parts: an ID and a secret, formatted as `<id>.<secret>`. The ID and secret are randomly generated strings that authenticate the joining nodes to the cluster.

The generated token is configured with specific permissions using RBAC policies. These permissions typically allow the token to create a certificate signing request (CSR) that the Kubernetes control plane can then approve, granting the joining node the necessary certificates to communicate securely within the cluster.

By default, bootstrap tokens are set to expire after a certain period (24 hours by default), ensuring that tokens cannot be reused indefinitely for joining new nodes to the cluster. This behavior enhances the security posture of the cluster by limiting the window during which a token is valid.

Once generated and configured, the bootstrap token is stored as a secret in the `kube-system` namespace.

### `kubelet-finalize`

This phase ensures that the kubelet is fully configured with the necessary settings to securely and effectively participate in the cluster. It involves applying any final kubelet configurations that might depend on the completion of the TLS bootstrap process.

### `addon`

This phase sets up essential add-ons required for the cluster to meet the Kubernetes Conformance Tests.

#### `/coredns`

CoreDNS provides DNS services for the internal cluster network, allowing pods to find each other by name and services to load-balance across a set of pods.

#### `/kube-proxy`

`kube-proxy` is responsible for managing network communication inside the cluster, implementing part of the Kubernetes Service concept by maintaining network rules on nodes. These rules allow network communication to pods from network sessions inside or outside the cluster.

`kube-proxy` ensures that the networking aspect of Kubernetes services is correctly handled, allowing for the routing of traffic to the appropriate destinations. It operates in the user space, and it can also run in `iptables` mode, where it manipulates rules to allow network traffic. This allows services to be exposed to the external network, load balances traffic to pods across the multiple instances, etc.

### `show-join-command`

This phase simplifies the process of expanding a Kubernetes cluster by generating bootstrap tokens and providing the necessary command to join additional nodes, whether they are worker nodes or additional control plane nodes.

In the next section, we'll actually bootstrap the cluster.
