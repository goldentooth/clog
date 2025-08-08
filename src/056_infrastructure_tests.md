# Infrastructure Test Framework Improvements

After running comprehensive tests across the cluster, we discovered several critical issues with our test framework that were masking real infrastructure problems. This chapter documents the systematic fixes we implemented to ensure our automated testing provides accurate health monitoring.

## The Initial Problem

When running `goldentooth test all`, multiple test failures appeared across different nodes:

```bash
PLAY RECAP *********************************************************************
bettley                    : ok=47   changed=0    unreachable=0    failed=1    skipped=3    rescued=0    ignored=2
cargyll                    : ok=47   changed=0    unreachable=0    failed=1    skipped=3    rescued=0    ignored=1
dalt                       : ok=47   changed=0    unreachable=0    failed=1    skipped=3    rescued=0    ignored=1
```

The challenge was determining whether these failures indicated real infrastructure issues or problems with the test framework itself.

## Root Cause Analysis

### 1. Kubernetes API Server Connectivity Issues

The most critical failure was the Kubernetes API server health check consistently failing on the `bettley` control plane node:

```
Status code was -1 and not [200]: Request failed: <urlopen error [Errno 111] Connection refused>
url: https://10.4.0.11:6443/healthz
```

Initial investigation revealed that while kubelet was running, both etcd and kube-apiserver pods were in CrashLoopBackOff state. This led us to discover that **Kubernetes certificates had expired** on June 20, 2025, but we were running tests in July 2025.

### 2. Test Framework Configuration Issues

Several test framework bugs were identified:

- **Vault decryption errors**: Tests couldn't access encrypted vault secrets
- **Wrong certificate paths**: Tests checking CA certificates instead of service certificates
- **Undefined variables**: JMESPath dependencies and variable reference errors
- **Localhost binding assumptions**: Services bound to specific IPs, not localhost

## Infrastructure Fixes

### Kubernetes Certificate Renewal

The most significant infrastructure issue was expired Kubernetes certificates. We resolved this using kubeadm:

```bash
# Backup existing certificates
ansible -i inventory/hosts bettley -m shell -a "cp -r /etc/kubernetes/pki /etc/kubernetes/pki.backup.$(date +%Y%m%d_%H%M%S)" --become

# Renew all certificates
ansible -i inventory/hosts bettley -m shell -a "kubeadm certs renew all" --become

# Restart control plane components by moving manifests temporarily
cd /etc/kubernetes/manifests
mv kube-apiserver.yaml kube-apiserver.yaml.tmp
mv etcd.yaml etcd.yaml.tmp
mv kube-controller-manager.yaml kube-controller-manager.yaml.tmp
mv kube-scheduler.yaml kube-scheduler.yaml.tmp

# Wait 10 seconds, then restore manifests
sleep 10
mv kube-apiserver.yaml.tmp kube-apiserver.yaml
mv etcd.yaml.tmp etcd.yaml
mv kube-controller-manager.yaml.tmp kube-controller-manager.yaml
mv kube-scheduler.yaml.tmp kube-scheduler.yaml
```

After renewal, certificates were valid until July 2026:

```
CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
apiserver                  Jul 23, 2026 00:01 UTC   364d            ca                      no
etcd-peer                  Jul 23, 2026 00:01 UTC   364d            etcd-ca                 no
etcd-server                Jul 23, 2026 00:01 UTC   364d            etcd-ca                 no
```

## Test Framework Fixes

### 1. Vault Authentication

Fixed missing vault password configuration in test environment:

```ini
# /Users/nathan/Projects/goldentooth/ansible/tests/ansible.cfg
[defaults]
vault_password_file = ~/.goldentooth_vault_password
```

### 2. Certificate Path Corrections

Updated tests to check actual service certificates instead of CA certificates:

```yaml
# Before: Checking CA certificates (5-year lifespan)
path: /etc/consul.d/tls/consul-agent-ca.pem

# After: Checking service certificates (24-hour rotation)
path: /etc/consul.d/certs/tls.crt
```

### 3. API Connectivity Fixes

Fixed hardcoded localhost assumptions to use actual node IP addresses:

```yaml
# Before: Assuming localhost binding
url: "https://127.0.0.1:8501/v1/status/leader"

# After: Using actual node IP
url: "http://{{ ansible_default_ipv4.address }}:8500/v1/status/leader"
```

### 4. Consul Members Command

Enhanced Consul connectivity testing with proper address specification:

```yaml
- name: Check if consul command exists
  stat:
    path: /usr/bin/consul
  register: consul_command_stat

- name: Check Consul members
  command: consul members -status=alive -http-addr={{ ansible_default_ipv4.address }}:8500
  when:
    - consul_service.status.ActiveState == "active"
    - consul_command_stat.stat.exists
```

### 5. Kubernetes Test Improvements

Simplified Kubernetes tests to avoid JMESPath dependencies and fixed variable scoping:

```yaml
# Simplified node readiness test
- name: Record node readiness test (simplified)
  set_fact:
    k8s_tests: "{{ k8s_tests + [{'name': 'k8s_cluster_accessible', 'category': 'kubernetes', 'success': (k8s_nodes_raw is defined and k8s_nodes_raw is succeeded) | bool, 'duration': 0.5}] }}"

# Fixed API health test scoping
- name: Record API health test
  set_fact:
    k8s_tests: "{{ k8s_tests + [{'name': 'k8s_api_healthy', 'category': 'kubernetes', 'success': (k8s_api.status == 200 and k8s_api.content | default('') == 'ok') | bool, 'duration': 0.2}] }}"
  when:
    - k8s_api is defined
    - inventory_hostname in groups['k8s_control_plane']
```

### 6. Step-CA Variable References

Fixed undefined variable references in Step-CA connectivity tests:

```yaml
# Fixed IP address lookup
elif step ca health --ca-url https://{{ hostvars[groups['step_ca'] | first]['ipv4_address'] }}:9443 --root /etc/ssl/certs/goldentooth.pem; then
```

### 7. Localhost Aggregation Task

Fixed the test summary task that was failing due to missing facts:

```yaml
- name: Aggregate test results
  hosts: localhost
  gather_facts: true  # Changed from false
```

## Test Design Philosophy

We adopted a principle of **separating certificate presence from validity testing**:

```yaml
# Test 1: Certificate exists
- name: Check Consul certificate exists
  stat:
    path: /etc/consul.d/certs/tls.crt
  register: consul_cert

- name: Record certificate presence test
  set_fact:
    consul_tests: "{{ consul_tests + [{'name': 'consul_certificate_present', 'category': 'consul', 'success': consul_cert.stat.exists | bool, 'duration': 0.1}] }}"

# Test 2: Certificate is valid (separate test)
- name: Check if certificate needs renewal
  command: step certificate needs-renewal /etc/consul.d/certs/tls.crt
  register: cert_needs_renewal
  when: consul_cert.stat.exists

- name: Record certificate validity test
  set_fact:
    consul_tests: "{{ consul_tests + [{'name': 'consul_certificate_valid', 'category': 'consul', 'success': (cert_needs_renewal.rc != 0) | bool, 'duration': 0.1}] }}"
```

This approach provides better debugging information and clearer failure isolation.
