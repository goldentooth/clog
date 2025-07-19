# Joining the Rest of the Control Plane

The next phase of bootstrapping is to admit the rest of the control plane nodes to the control plane.

## Certificate Key Extraction

Before joining additional control plane nodes, we need to extract the certificate key from the initial `kubeadm init` output:

```yaml
- name: 'Set the kubeadm certificate key.'
  ansible.builtin.set_fact:
    k8s_certificate_key: "{{ line | regex_search('--certificate-key ([^ ]+)', '\\1') | first }}"
  loop: "{{ hostvars[kubernetes.first]['kubeadm_init'].stdout_lines | default([]) }}"
  when: '(line | trim) is match(".*--certificate-key.*")'
```

This certificate key is crucial for securely downloading control plane certificates during the join process. The `--upload-certs` flag from the initial `kubeadm init` uploaded these certificates to the cluster, encrypted with this key.

## Dynamic Token Generation

Rather than using a static token, we generate a fresh token for the join process:

```yaml
- name: 'Create kubeadm token for joining nodes.'
  ansible.builtin.command:
    cmd: "kubeadm --kubeconfig {{ kubernetes.admin_conf_path }} token create"
  delegate_to: "{{ kubernetes.first }}"
  register: 'temp_token'

- name: 'Set kubeadm token fact.'
  ansible.builtin.set_fact:
    kubeadm_token: "{{ temp_token.stdout }}"
```

This approach:
- **Security**: Uses short-lived tokens (24-hour default expiry)
- **Automation**: No need to manually specify or distribute tokens
- **Reliability**: Fresh token for each bootstrap operation

## JoinConfiguration Template

The JoinConfiguration manifest is generated from a Jinja2 template (`kubeadm-controlplane.yaml.j2`):

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: JoinConfiguration
discovery:
  bootstrapToken:
    apiServerEndpoint: {{ haproxy.ipv4_address }}:6443
    token: {{ kubeadm_token }}
    unsafeSkipCAVerification: true
  timeout: 5m0s
  tlsBootstrapToken: {{ kubeadm_token }}
controlPlane:
  localAPIEndpoint:
    advertiseAddress: {{ ipv4_address }}
    bindPort: 6443
  certificateKey: {{ k8s_certificate_key }}
nodeRegistration:
  name: {{ inventory_hostname }}
  criSocket: {{ kubernetes.cri_socket_path }}
{% if inventory_hostname in kubernetes.rest %}
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/control-plane
{% else %}
  taints: []
{% endif %}
```

**Key Configuration Elements**:

**Discovery Configuration**:
- **API Server Endpoint**: Points to HAProxy load balancer (`10.4.0.10:6443`)
- **Bootstrap Token**: Dynamically generated token for secure cluster discovery
- **CA Verification**: Skipped for simplicity (acceptable in trusted network)
- **Timeout**: 5-minute timeout for discovery operations

**Control Plane Configuration**:
- **Local API Endpoint**: Each node advertises its own IP for API server communication
- **Certificate Key**: Enables secure download of control plane certificates
- **Bind Port**: Standard Kubernetes API server port (6443)

**Node Registration**:
- **CRI Socket**: Uses containerd socket (`unix:///var/run/containerd/containerd.sock`)
- **Node Name**: Uses Ansible inventory hostname for consistency
- **Taints**: Control plane nodes get NoSchedule taint to prevent workload scheduling

## Control Plane Join Process

The actual joining process involves several orchestrated steps:

### 1. Configuration Setup

```yaml
- name: 'Ensure presence of Kubernetes directory.'
  ansible.builtin.file:
    path: '/etc/kubernetes'
    state: 'directory'
    mode: '0755'

- name: 'Create kubeadm control plane config.'
  ansible.builtin.template:
    src: 'kubeadm-controlplane.yaml.j2'
    dest: '/etc/kubernetes/kubeadm-controlplane.yaml'
    mode: '0640'
    backup: true
```

### 2. Readiness Verification

```yaml
- name: 'Wait for the kube-apiserver to be ready.'
  ansible.builtin.wait_for:
    host: "{{ haproxy.ipv4_address }}"
    port: '6443'
    timeout: 180
```

This ensures the load balancer and initial control plane node are ready before attempting to join.

### 3. Clean State Reset

```yaml
- name: 'Reset certificate directory.'
  ansible.builtin.shell:
    cmd: |
      if [ -f /etc/kubernetes/manifests/kube-apiserver.yaml ]; then
        kubeadm reset -f --cert-dir {{ kubernetes.pki_path }};
      fi
```

This conditional reset ensures a clean state if a node was previously part of a cluster.

### 4. Control Plane Join

```yaml
- name: 'Join the control plane node to the cluster.'
  ansible.builtin.command:
    cmd: kubeadm join --config /etc/kubernetes/kubeadm-controlplane.yaml
  register: 'kubeadm_join'
```

### 5. Administrative Access Setup

```yaml
- name: 'Ensure .kube directory exists.'
  ansible.builtin.file:
    path: '~/.kube'
    state: 'directory'
    mode: '0755'

- name: 'Symlink the kubectl admin.conf to ~/.kube/config.'
  ansible.builtin.file:
    src: '/etc/kubernetes/admin.conf'
    dest: '~/.kube/config'
    state: 'link'
    mode: '0600'
```

This sets up `kubectl` access for the root user on each control plane node.

## Target Nodes

The control plane joining process targets nodes in the `kubernetes.rest` group:
- **bettley** (10.4.0.11) - Second control plane node
- **cargyll** (10.4.0.12) - Third control plane node

This gives us a 3-node control plane for high availability, capable of surviving the failure of any single node.

## High Availability Considerations

**Load Balancer Integration**:
- All control plane nodes use the HAProxy endpoint for cluster communication
- Even control plane nodes connect through the load balancer for API server access
- This ensures consistent behavior whether accessing from inside or outside the cluster

**Certificate Management**:
- Control plane certificates are automatically distributed via the certificate key mechanism
- Each node gets its own API server certificate with appropriate SANs
- Certificate rotation is handled through normal Kubernetes certificate management

**Etcd Clustering**:
- kubeadm automatically configures etcd clustering across all control plane nodes
- Stacked topology (etcd on same nodes as API server) for simplicity
- Quorum maintained with 3 nodes (can survive 1 node failure)

After these steps complete, a simple `kubeadm join --config /etc/kubernetes/kubeadm-controlplane.yaml` on each remaining control plane node is sufficient to complete the highly available control plane setup.
