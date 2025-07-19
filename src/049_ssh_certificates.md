# SSH Certificates

So remember back in [chapter 32](032_more_pki_with_step_ca.md) when I set up Step-CA as our internal certificate authority? Step-CA also handle SSH certificates, which allows a less peer-to-peer model for authenticating between nodes. I'd actually tried to set these up before and it was an _enormous_ pain in the pass and didn't really work well, so when I saw Step-CA included it in its featureset, I was excited.

It's very easy to allow `authorized_keys` to grow without bound, and I'm fairly sure very few people actually read these messages:

```
The authenticity of host 'wtf.node.goldentooth.net (192.168.10.51)' can't be established.
ED25519 key fingerprint is SHA256:8xKJ5Fw6K+YFGxqR5EWsM4w3t5Y7MzO1p3G9kPvXHDo.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

So I wanted something that would allow seamless interconnection between the nodes while maintaining good security.

SSH certificates solve both of these problems elegantly. Instead of managing individual keys, you have a certificate authority that signs certificates. For user authentication, the SSH server trusts the CA's public key. For host authentication, your SSH client trusts the CA's public key.

It's basically the same model as TLS certificates, but for SSH. And since we already have Step-CA running, why not use it?

## The Implementation

I created an Ansible role called `goldentooth.setup_ssh_certificates` to handle all of this. Let me walk through what it does.

### Setting Up the CA Trust

First, we need to grab the SSH CA public keys from our Step-CA server. There are actually two different keys - one for signing user certificates and one for signing host certificates:

```yaml
- name: 'Get SSH User CA public key'
  ansible.builtin.slurp:
    src: "{{ step_ca.ca.etc_path }}/certs/ssh_user_ca_key.pub"
  register: 'ssh_user_ca_key_b64'
  delegate_to: "{{ step_ca.server }}"
  run_once: true
  become: true

- name: 'Get SSH Host CA public key'
  ansible.builtin.slurp:
    src: "{{ step_ca.ca.etc_path }}/certs/ssh_host_ca_key.pub"
  register: 'ssh_host_ca_key_b64'
  delegate_to: "{{ step_ca.server }}"
  run_once: true
  become: true
```

Then we configure sshd to trust certificates signed by our User CA:

```yaml
- name: 'Configure sshd to trust User CA'
  ansible.builtin.lineinfile:
    path: '/etc/ssh/sshd_config'
    regexp: '^#?TrustedUserCAKeys'
    line: 'TrustedUserCAKeys /etc/ssh/ssh_user_ca.pub'
    state: 'present'
    validate: '/usr/sbin/sshd -t -f %s'
  notify: 'reload sshd'
```

### Host Certificates

For host certificates, we generate a certificate for each node that includes multiple principals (names the certificate is valid for):

```yaml
- name: 'Generate SSH host certificate'
  ansible.builtin.shell:
    cmd: |
      step ssh certificate \
        --host \
        --sign \
        --force \
        --no-password \
        --insecure \
        --provisioner="{{ step_ca.default_provisioner.name }}" \
        --provisioner-password-file="{{ step_ca.default_provisioner.password_path }}" \
        --principal="{{ ansible_hostname }}" \
        --principal="{{ ansible_hostname }}.{{ cluster.node_domain }}" \
        --principal="{{ ansible_hostname }}.{{ cluster.domain }}" \
        --principal="{{ ansible_default_ipv4.address }}" \
        --ca-url="https://{{ hostvars[step_ca.server].ipv4_address }}:9443" \
        --root="{{ step_ca.root_cert_path }}" \
        --not-after=24h \
        {{ ansible_hostname }} \
        /etc/step/certs/ssh_host.key.pub
```

### Automatic Certificate Renewal

Notice the `--not-after=24h`? Yeah, these certificates expire daily. Which means it's very important that the automatic renewal works 😀

Enter systemd timers:

```ini
[Unit]
Description=Timer for SSH host certificate renewal
Documentation=https://smallstep.com/docs/step-cli/reference/ssh/certificate

[Timer]
OnBootSec=5min
OnUnitActiveSec=15min
RandomizedDelaySec=5min

[Install]
WantedBy=timers.target
```

This runs every 15 minutes (with some randomization to avoid thundering herd problems). The service itself checks if the certificate needs renewal before actually doing anything:

```ini
# Check if certificate needs renewal
ExecCondition=/usr/bin/step certificate needs-renewal /etc/step/certs/ssh_host.key-cert.pub
```

### User Certificates

For user certificates, I set up both root and my regular user account. The process is similar - generate a certificate with appropriate principals:

```yaml
- name: 'Generate root user SSH certificate'
  ansible.builtin.shell:
    cmd: |
      step ssh certificate \
        --sign \
        --force \
        --no-password \
        --insecure \
        --provisioner="{{ step_ca.default_provisioner.name }}" \
        --provisioner-password-file="{{ step_ca.default_provisioner.password_path }}" \
        --principal="root" \
        --principal="{{ ansible_hostname }}-root" \
        --ca-url="https://{{ hostvars[step_ca.server].ipv4_address }}:9443" \
        --root="{{ step_ca.root_cert_path }}" \
        --not-after=24h \
        root@{{ ansible_hostname }} \
        /etc/step/certs/root_ssh_key.pub
```

Then configure SSH to actually use the certificate:

```yaml
- name: 'Configure root SSH to use certificate'
  ansible.builtin.blockinfile:
    path: '/root/.ssh/config'
    create: true
    owner: 'root'
    group: 'root'
    mode: '0600'
    block: |
      Host *
          CertificateFile /etc/step/certs/root_ssh_key-cert.pub
          IdentityFile /etc/step/certs/root_ssh_key
    marker: '# {mark} ANSIBLE MANAGED BLOCK - SSH CERTIFICATE'
```

### The Trust Configuration

For the client side, we need to tell SSH to trust host certificates signed by our CA:

```yaml
- name: 'Configure SSH client to trust Host CA'
  ansible.builtin.lineinfile:
    path: '/etc/ssh/ssh_known_hosts'
    line: "@cert-authority * {{ ssh_host_ca_key }}"
    create: true
    owner: 'root'
    group: 'root'
    mode: '0644'
```

And since we're all friends here in the cluster, I disabled strict host key checking for cluster nodes:

```yaml
- name: 'Disable StrictHostKeyChecking for cluster nodes'
  ansible.builtin.blockinfile:
    path: '/etc/ssh/ssh_config'
    block: |
      Host *.{{ cluster.node_domain }} *.{{ cluster.domain }}
          StrictHostKeyChecking no
          UserKnownHostsFile /dev/null
    marker: '# {mark} ANSIBLE MANAGED BLOCK - CLUSTER SSH CONFIG'
```

Is this less secure? Technically yes. Do I care? Not really. These are all nodes in my internal cluster that I control. The certificates provide the actual authentication.

## The Results

After running the playbook, I can now SSH between any nodes in the cluster without passwords or key management:

```bash
root@bramble-ca:~# ssh bramble-01
Welcome to Ubuntu 24.04.1 LTS (GNU/Linux 6.8.0-1017-raspi aarch64)
...
Last login: Sat Jul 19 00:15:23 2025 from 192.168.10.50
root@bramble-01:~#
```

No host key verification prompts. No password prompts. Just instant access.

And the best part? I can verify that certificates are being used:

```bash
root@bramble-01:~# ssh-keygen -L -f /etc/step/certs/ssh_host.key-cert.pub
/etc/step/certs/ssh_host.key-cert.pub:
        Type: ssh-ed25519-cert-v01@openssh.com host certificate
        Public key: ED25519-CERT SHA256:M5PQn6zVH7xJL+OFQzH4yVwR5EHrF2xQPm9QR5xKXBc
        Signing CA: ED25519 SHA256:gNPpOqPsZW6YZDmhWQWqJ4l+L8E5Xgg8FQyAAbPi7Ss (using ssh-ed25519)
        Key ID: "bramble-01"
        Serial: 8485811653946933657
        Valid: from 2025-07-18T20:13:42 to 2025-07-19T20:14:42
        Principals:
                bramble-01
                bramble-01.node.goldentooth.net
                bramble-01.goldentooth.net
                192.168.10.51
        Critical Options: (none)
        Extensions: (none)
```

Look at that! The certificate is valid for exactly 24 hours and includes all the names I might use to connect to this host.

## Lessons Learned

This whole setup taught me a few things:

1. **SSH certificates are underutilized** - This technology has been around for years but hardly anyone uses it. It's a shame because it solves real problems.

2. **Short-lived certificates are the way** - Daily renewal might seem extreme, but with automation it's invisible. And the security benefits are real.

3. **Step-CA continues to impress** - Every time I dig deeper into this tool, I find more useful features. The fact that it handles both TLS and SSH certificates with the same infrastructure is brilliant.

4. **Ansible makes this manageable** - Setting this up manually on each node would be a nightmare. With Ansible, it's just `ansible-playbook playbooks/setup_ssh_certificates.yaml` and done.

## Future Improvements

There are a few things I might add later:

- **Certificate templates** - Step-CA supports templates that can add more complex logic to certificate issuance
- **Hardware key support** - You can store SSH certificates on hardware tokens like YubiKeys
- **Audit logging** - Every certificate issuance is logged by Step-CA, which could feed into our observability stack
- **User provisioning** - Right now I'm only handling root and my user. A more complete solution would handle arbitrary users

But for now, this works great. No more SSH key management, no more host key warnings, just secure, certificate-based authentication across the cluster.
