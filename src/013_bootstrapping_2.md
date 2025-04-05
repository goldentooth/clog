# Joining the Rest of the Control Plane

The next phase of bootstrapping is to admit the rest of the control plane nodes to the control plane.

First, we create a JoinConfiguration manifest, which should look something like this (in Jinja2):

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: JoinConfiguration
discovery:
  bootstrapToken:
    apiServerEndpoint: {{ load_balancer.ipv4_address }}:6443
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
  criSocket: {{ k8s_cri_socket }}
{% if inventory_hostname in control_plane.rest.hostnames %}
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/control-plane
{% else %}
  taints: []
{% endif %}
```

I haven't bothered to substitute the values; none of them should be mysterious at this point.

After that, a simple `kubeadm join --config /etc/kubernetes/kubeadm-controlplane.yaml` on each node is sufficient to complete the control plane.
