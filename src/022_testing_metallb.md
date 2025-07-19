# Testing MetalLB

The simplest way to test MetalLB is just to deploy an application with a `LoadBalancer` service and see if it works.

I'm a fan of `httpbin` and its Go port, [`httpbingo`](https://httpbingo.org), so up it goes:

```yaml
apiVersion: 'argoproj.io/v1alpha1'
kind: 'Application'
metadata:
  name: 'httpbin'
  namespace: 'argocd'
  labels:
    name: 'httpbin'
    managed-by: 'argocd'
spec:
  project: 'httpbin'
  source:
    repoURL: 'https://matheusfm.dev/charts'
    chart: 'httpbin'
    targetRevision: '0.1.1'
    helm:
      releaseName: 'httpbin'
      valuesObject:
        service:
          type: 'LoadBalancer'
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: 'httpbin'
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - Validate=true
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
      - RespectIgnoreDifferences=true
      - ApplyOutOfSyncOnly=true
```

Very quickly, it's synced:

![httpbin deployed](./images/022_httpbin_synced.png)

We can get the IP address allocated for the load balancer with `kubectl -n httpbin get svc`:

![httpbin service](./images/022_httpbin_service.png)

And sure enough, it's allocated from the IP address pool we specified. That seems like an excellent sign!

Can we access it from a web browser running on a computer on a different network?

![httpbin webpage](./images/022_httpbin_page.png)

Yes, we can! Our load balancer system is working!

## Comprehensive MetalLB Testing Suite

While the httpbin test demonstrates basic functionality, production MetalLB deployments require more thorough validation of various scenarios and failure modes.

### Phase 1: Basic Functionality Tests

#### 1.1 IP Address Allocation Verification

First, verify that MetalLB allocates IP addresses from the configured pool:

```bash
# Check the configured IP address pool
kubectl -n metallb get ipaddresspool primary -o yaml

# Deploy multiple LoadBalancer services and verify allocations
kubectl create deployment test-nginx --image=nginx
kubectl expose deployment test-nginx --type=LoadBalancer --port=80

# Verify sequential allocation from pool
kubectl get svc test-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

Expected behavior:
- IPs allocated from `10.4.11.0 - 10.4.15.254` range
- Sequential allocation starting from pool beginning
- No IP conflicts between services

#### 1.2 Service Lifecycle Testing

Test the complete service lifecycle to ensure proper cleanup:

```bash
# Create service and note allocated IP
kubectl create deployment lifecycle-test --image=httpd
kubectl expose deployment lifecycle-test --type=LoadBalancer --port=80
ALLOCATED_IP=$(kubectl get svc lifecycle-test -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Verify service is accessible
curl -s http://$ALLOCATED_IP/ | grep "It works!"

# Delete service and verify IP is released
kubectl delete svc lifecycle-test
kubectl delete deployment lifecycle-test

# Verify IP is available for reallocation
kubectl create deployment reallocation-test --image=nginx
kubectl expose deployment reallocation-test --type=LoadBalancer --port=80
NEW_IP=$(kubectl get svc reallocation-test -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Should reuse the previously released IP
echo "Original IP: $ALLOCATED_IP, New IP: $NEW_IP"
```

### Phase 2: BGP Advertisement Testing

#### 2.1 BGP Session Health Verification

Monitor BGP session establishment and health:

```bash
# Check MetalLB speaker status
kubectl -n metallb get pods -l component=speaker

# Verify BGP sessions from router perspective
goldentooth command allyrion 'vtysh -c "show ip bgp summary"'

# Check BGP neighbor status for specific node
goldentooth command allyrion 'vtysh -c "show ip bgp neighbor 10.4.0.11"'
```

Expected BGP session states:
- **Established**: BGP session is active and exchanging routes
- **Route count**: Number of routes received from each speaker
- **Session uptime**: Indicates session stability

#### 2.2 Route Advertisement Verification

Verify that LoadBalancer IPs are properly advertised via BGP:

```bash
# Create test service
kubectl create deployment bgp-test --image=nginx
kubectl expose deployment bgp-test --type=LoadBalancer --port=80
TEST_IP=$(kubectl get svc bgp-test -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Check route advertisement on router
goldentooth command allyrion "vtysh -c 'show ip bgp | grep $TEST_IP'"

# Verify route in kernel routing table
goldentooth command allyrion "ip route show | grep $TEST_IP"

# Test route withdrawal
kubectl delete svc bgp-test
sleep 30

# Verify route is withdrawn
goldentooth command allyrion "vtysh -c 'show ip bgp | grep $TEST_IP' || echo 'Route withdrawn'"
```

### Phase 3: High Availability Testing

#### 3.1 Speaker Node Failure Simulation

Test MetalLB's behavior when speaker nodes fail:

```bash
# Identify which node is announcing a service
kubectl create deployment ha-test --image=nginx
kubectl expose deployment ha-test --type=LoadBalancer --port=80
HA_IP=$(kubectl get svc ha-test -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Find announcing node from BGP table
goldentooth command allyrion "vtysh -c 'show ip bgp $HA_IP'"

# Simulate node failure by stopping kubelet on announcing node
ANNOUNCING_NODE=$(kubectl get svc ha-test -o jsonpath='{.metadata.annotations.metallb\.universe\.tf/announcing-node}' 2>/dev/null || echo "bettley")
goldentooth command_root $ANNOUNCING_NODE 'systemctl stop kubelet'

# Wait for BGP reconvergence (typically 30-180 seconds)
sleep 60

# Verify service is still accessible (new node should announce)
curl -s http://$HA_IP/ | grep "Welcome to nginx"

# Check new announcing node
goldentooth command allyrion "vtysh -c 'show ip bgp $HA_IP'"

# Restore failed node
goldentooth command_root $ANNOUNCING_NODE 'systemctl start kubelet'
```

#### 3.2 Split-Brain Prevention Testing

Verify that MetalLB prevents split-brain scenarios where multiple nodes announce the same service:

```bash
# Deploy service with specific node selector to control placement
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: split-brain-test
  annotations:
    metallb.universe.tf/allow-shared-ip: "split-brain-test"
spec:
  type: LoadBalancer
  selector:
    app: split-brain-test
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: split-brain-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: split-brain-test
  template:
    metadata:
      labels:
        app: split-brain-test
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# Monitor BGP announcements for the service IP
SPLIT_IP=$(kubectl get svc split-brain-test -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
goldentooth command allyrion "vtysh -c 'show ip bgp $SPLIT_IP detail'"

# Should see only one announcement path, not multiple conflicting paths
```

### Phase 4: Performance and Scale Testing

#### 4.1 IP Pool Exhaustion Testing

Test behavior when IP address pool is exhausted:

```bash
# Calculate available IPs in pool (10.4.11.0 - 10.4.15.254 = ~1250 IPs)
# Deploy services until pool exhaustion

for i in {1..10}; do
  kubectl create deployment scale-test-$i --image=nginx
  kubectl expose deployment scale-test-$i --type=LoadBalancer --port=80
  echo "Created service $i"
  sleep 5
done

# Check for services stuck in Pending state
kubectl get svc | grep Pending

# Verify MetalLB events for pool exhaustion
kubectl -n metallb get events --sort-by='.lastTimestamp'
```

#### 4.2 BGP Convergence Time Measurement

Measure BGP convergence time under various scenarios:

```bash
# Create test service and measure initial advertisement time
start_time=$(date +%s)
kubectl create deployment convergence-test --image=nginx
kubectl expose deployment convergence-test --type=LoadBalancer --port=80

# Wait for IP allocation
while [ -z "$(kubectl get svc convergence-test -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null)" ]; do
  sleep 1
done

CONV_IP=$(kubectl get svc convergence-test -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "IP allocated: $CONV_IP"

# Wait for BGP advertisement
while ! goldentooth command allyrion "ip route show | grep $CONV_IP" >/dev/null 2>&1; do
  sleep 1
done

end_time=$(date +%s)
convergence_time=$((end_time - start_time))
echo "BGP convergence time: ${convergence_time} seconds"
```

### Phase 5: Integration Testing

#### 5.1 ExternalDNS Integration

Test automatic DNS record creation for LoadBalancer services:

```bash
# Deploy service with DNS annotation
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: dns-integration-test
  annotations:
    external-dns.alpha.kubernetes.io/hostname: test.goldentooth.net
spec:
  type: LoadBalancer
  selector:
    app: dns-test
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dns-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dns-test
  template:
    metadata:
      labels:
        app: dns-test
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# Wait for DNS propagation
sleep 60

# Test DNS resolution
nslookup test.goldentooth.net

# Test HTTP access via DNS name
curl -s http://test.goldentooth.net/ | grep "Welcome to nginx"
```

#### 5.2 TLS Certificate Integration

Test automatic TLS certificate provisioning for LoadBalancer services:

```bash
# Deploy service with cert-manager annotations
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: tls-integration-test
  annotations:
    external-dns.alpha.kubernetes.io/hostname: tls-test.goldentooth.net
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  type: LoadBalancer
  selector:
    app: tls-test
  ports:
  - port: 443
    targetPort: 443
    name: https
  - port: 80
    targetPort: 80
    name: http
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-test-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - tls-test.goldentooth.net
    secretName: tls-test-cert
  rules:
  - host: tls-test.goldentooth.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tls-integration-test
            port:
              number: 80
EOF

# Wait for certificate provisioning
kubectl wait --for=condition=Ready certificate/tls-test-cert --timeout=300s

# Test HTTPS access
curl -s https://tls-test.goldentooth.net/ | grep "Welcome to nginx"
```

### Phase 6: Troubleshooting and Monitoring

#### 6.1 MetalLB Component Health

Monitor MetalLB component health and logs:

```bash
# Check MetalLB controller status
kubectl -n metallb get pods -l component=controller
kubectl -n metallb logs -l component=controller --tail=50

# Check MetalLB speaker status on each node
kubectl -n metallb get pods -l component=speaker -o wide
kubectl -n metallb logs -l component=speaker --tail=50

# Check MetalLB configuration
kubectl -n metallb get ipaddresspool,bgppeer,bgpadvertisement -o wide
```

#### 6.2 BGP Session Troubleshooting

Debug BGP session issues:

```bash
# Check BGP session state
goldentooth command allyrion 'vtysh -c "show ip bgp summary"'

# Detailed neighbor analysis
for node_ip in 10.4.0.11 10.4.0.12 10.4.0.13; do
  echo "=== BGP Neighbor $node_ip ==="
  goldentooth command allyrion "vtysh -c 'show ip bgp neighbor $node_ip'"
done

# Check for BGP route-map and prefix-list configurations
goldentooth command allyrion 'vtysh -c "show ip bgp route-map"'
goldentooth command allyrion 'vtysh -c "show ip prefix-list"'

# Monitor BGP route changes in real-time
goldentooth command allyrion 'vtysh -c "debug bgp events"'
```

#### 6.3 Network Connectivity Testing

Comprehensive network path testing:

```bash
# Test connectivity from external networks
for test_ip in $(kubectl get svc -A -o jsonpath='{.items[?(@.spec.type=="LoadBalancer")].status.loadBalancer.ingress[0].ip}'); do
  echo "Testing connectivity to $test_ip"
  
  # Test from router
  goldentooth command allyrion "ping -c 3 $test_ip"
  
  # Test HTTP connectivity
  goldentooth command allyrion "curl -s -o /dev/null -w '%{http_code}' http://$test_ip/ || echo 'Connection failed'"
  
  # Test from external network (if possible)
  # ping -c 3 $test_ip
done

# Test internal cluster connectivity
kubectl run network-test --image=busybox --rm -it --restart=Never -- /bin/sh
# From within the pod:
# wget -qO- http://test-service.default.svc.cluster.local/
```

This comprehensive testing suite ensures MetalLB is functioning correctly across all operational scenarios, from basic IP allocation to complex failure recovery and integration testing. Each test phase builds confidence in the load balancer implementation and helps identify potential issues before they impact production workloads.
