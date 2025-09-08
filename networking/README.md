# Networking Troubleshooting Guide

Guide for troubleshooting Kubernetes networking, DNS, and connectivity issues.

## Common Network Issues

### Pod-to-Pod Communication Problems

```bash
# Test connectivity between pods
kubectl exec -it <source-pod> -n <namespace> -- ping <target-pod-ip>
kubectl exec -it <source-pod> -n <namespace> -- telnet <target-service> <port>

# Check network policies
kubectl get networkpolicies -n <namespace>
kubectl describe networkpolicy <policy-name> -n <namespace>

# Check CNI plugin status
kubectl get pods -n kube-system | grep -E "(calico|weave|flannel|cilium)"
kubectl logs -n kube-system <cni-pod>

# Check iptables rules (on nodes)
sudo iptables -L -n
```

### DNS Resolution Issues

```bash
# Test DNS resolution
kubectl exec -it <pod-name> -n <namespace> -- nslookup kubernetes.default.svc.cluster.local
kubectl exec -it <pod-name> -n <namespace> -- cat /etc/resolv.conf

# Check CoreDNS
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system <coredns-pod>
kubectl get configmap coredns -n kube-system -o yaml

# Check service discovery
kubectl get svc -n kube-system | grep dns
```

### Network Policy Issues

```bash
# Check applied network policies
kubectl get networkpolicies --all-namespaces
kubectl describe networkpolicy <policy-name> -n <namespace>

# Test connectivity without policies
kubectl label namespace <namespace> name=<namespace>
# Temporarily disable policies for testing

# Check policy selectors
kubectl get pods -n <namespace> --show-labels
```

## Quick Commands

```bash
# Network testing
kubectl exec -it <pod-name> -n <namespace> -- ping <target>
kubectl exec -it <pod-name> -n <namespace> -- nc -zv <host> <port>
kubectl exec -it <pod-name> -n <namespace> -- nslookup <service-name>

# DNS troubleshooting
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system <coredns-pod>
kubectl get configmap coredns -n kube-system -o yaml

# Network policies
kubectl get networkpolicies --all-namespaces
kubectl describe networkpolicy <policy-name> -n <namespace>

# CNI troubleshooting
kubectl get pods -n kube-system | grep -E "calico|weave|flannel|cilium"
kubectl logs -n kube-system <cni-pod>
```

---

## Hands-On Testing Scenarios

Practice these scenarios in your Minikube or kind cluster to gain practical Networking troubleshooting experience.

### Scenario 1: Pod-to-Pod Communication Issues

**Setup the Problem:**

```bash
# Create two pods in the same namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: client-pod
  labels:
    app: client
spec:
  containers:
  - name: client
    image: busybox
    command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: server-pod
  labels:
    app: server
spec:
  containers:
  - name: server
    image: nginx:alpine
    ports:
    - containerPort: 80
EOF

kubectl wait --for=condition=Ready pod/client-pod --timeout=60s
kubectl wait --for=condition=Ready pod/server-pod --timeout=60s

# Get server pod IP
SERVER_IP=$(kubectl get pod server-pod -o jsonpath='{.status.podIP}')
echo "Server Pod IP: $SERVER_IP"
```

**Troubleshoot:**

```bash
# Test basic connectivity
kubectl exec client-pod -- ping -c 3 $SERVER_IP

# Test port connectivity
kubectl exec client-pod -- nc -zv $SERVER_IP 80

# Test HTTP connection
kubectl exec client-pod -- wget -qO- http://$SERVER_IP

# Check pod network configuration
kubectl exec client-pod -- ip addr
kubectl exec server-pod -- ip addr

# Check routing
kubectl exec client-pod -- ip route
```

**Fix (if issues found):**

```bash
# Check CNI configuration
kubectl get nodes -o wide
kubectl describe node

# Check CNI pods
kubectl get pods -n kube-system | grep -E "calico|weave|flannel|cilium|kube-proxy"

# Restart CNI pods if needed
kubectl delete pods -n kube-system -l k8s-app=kube-proxy
```

### Scenario 2: Service Discovery and DNS Issues

**Setup the Problem:**

```bash
# Create service and deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web-server
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
---
apiVersion: v1
kind: Pod
metadata:
  name: dns-test-pod
spec:
  containers:
  - name: test
    image: busybox
    command: ["sleep", "3600"]
EOF

kubectl wait --for=condition=Ready pod/dns-test-pod --timeout=60s
kubectl wait --for=condition=Available deployment/web-server --timeout=60s
```

**Troubleshoot:**

```bash
# Test service DNS resolution
kubectl exec dns-test-pod -- nslookup web-service

# Test full DNS name
kubectl exec dns-test-pod -- nslookup web-service.default.svc.cluster.local

# Check if service has endpoints
kubectl get endpoints web-service

# Test service connectivity
kubectl exec dns-test-pod -- wget -qO- http://web-service

# Check DNS configuration
kubectl exec dns-test-pod -- cat /etc/resolv.conf

# Test external DNS
kubectl exec dns-test-pod -- nslookup google.com

# Check CoreDNS
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

**Fix (if DNS issues found):**

```bash
# Check CoreDNS configuration
kubectl get configmap coredns -n kube-system -o yaml

# Restart CoreDNS
kubectl delete pods -n kube-system -l k8s-app=kube-dns

# Check kube-dns service
kubectl get service -n kube-system kube-dns
```

### Scenario 3: NetworkPolicy Blocking Traffic

**Setup the Problem:**

```bash
# Create namespaces
kubectl create namespace production
kubectl create namespace development

# Create pods in both namespaces
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: prod-web
  namespace: production
  labels:
    app: web
    tier: production
spec:
  containers:
  - name: web
    image: nginx:alpine
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: prod-web-service
  namespace: production
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: dev-client
  namespace: development
spec:
  containers:
  - name: client
    image: busybox
    command: ["sleep", "3600"]
EOF

kubectl wait --for=condition=Ready pod/prod-web -n production --timeout=60s
kubectl wait --for=condition=Ready pod/dev-client -n development --timeout=60s

# Test initial connectivity (should work)
kubectl exec dev-client -n development -- wget -qO- --timeout=5 http://prod-web-service.production.svc.cluster.local

# Apply restrictive network policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: production
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          environment: production
EOF
```

**Troubleshoot:**

```bash
# Test connectivity (should now fail)
kubectl exec dev-client -n development -- wget -qO- --timeout=5 http://prod-web-service.production.svc.cluster.local || echo "Connection blocked"

# Check network policies
kubectl get networkpolicies -n production
kubectl describe networkpolicy deny-cross-namespace -n production

# Check namespace labels
kubectl get namespaces --show-labels

# Test internal namespace connectivity
kubectl run internal-test -n production --image=busybox --rm -it --restart=Never -- wget -qO- http://prod-web-service.production.svc.cluster.local
```

**Fix:**

```bash
# Option 1: Label development namespace appropriately
kubectl label namespace development environment=development

# Option 2: Modify network policy to allow development namespace
kubectl patch networkpolicy deny-cross-namespace -n production -p '{"spec":{"ingress":[{"from":[{"namespaceSelector":{"matchLabels":{"environment":"production"}}},{"namespaceSelector":{"matchLabels":{"environment":"development"}}}]}]}}'

# Option 3: Add specific pod selector exception
kubectl patch networkpolicy deny-cross-namespace -n production -p '{"spec":{"ingress":[{"from":[{"namespaceSelector":{"matchLabels":{"environment":"production"}}},{"podSelector":{"matchLabels":{"app":"allowed-client"}}}]}]}}'

# Test connectivity again
kubectl exec dev-client -n development -- wget -qO- http://prod-web-service.production.svc.cluster.local
```

### Scenario 4: Service Endpoint Issues

**Setup the Problem:**

```bash
# Create service with mismatched selector
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: broken-service
spec:
  selector:
    app: wrong-label  # This doesn't match any pods
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: endpoint-test-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: endpoint-test  # Different from service selector
  template:
    metadata:
      labels:
        app: endpoint-test
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF

kubectl wait --for=condition=Available deployment/endpoint-test-deployment --timeout=60s
```

**Troubleshoot:**

```bash
# Check service endpoints
kubectl get endpoints broken-service

# Compare with service selector
kubectl describe service broken-service

# Check deployment labels
kubectl get pods -l app=endpoint-test

# Test service connectivity
kubectl run test-client --image=busybox --rm -it --restart=Never -- wget -qO- --timeout=5 http://broken-service || echo "No endpoints available"

# Check service creation events
kubectl get events --sort-by=.metadata.creationTimestamp | grep broken-service
```

**Fix:**

```bash
# Fix by updating service selector
kubectl patch service broken-service -p '{"spec":{"selector":{"app":"endpoint-test"}}}'

# Verify endpoints are now populated
kubectl get endpoints broken-service

# Test connectivity
kubectl run test-client --image=busybox --rm -it --restart=Never -- wget -qO- http://broken-service
```

### Scenario 5: Port Configuration Issues

**Setup the Problem:**

```bash
# Create service with wrong port configuration
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: port-test-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: port-test
  template:
    metadata:
      labels:
        app: port-test
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: port-test-service
spec:
  selector:
    app: port-test
  ports:
  - port: 80
    targetPort: 8080  # Wrong target port
EOF

kubectl wait --for=condition=Available deployment/port-test-deployment --timeout=60s
```

**Troubleshoot:**

```bash
# Test service connectivity (should fail)
kubectl run test-client --image=busybox --rm -it --restart=Never -- wget -qO- --timeout=5 http://port-test-service || echo "Connection failed"

# Check service configuration
kubectl describe service port-test-service

# Check pod ports
kubectl describe pod -l app=port-test

# Get endpoint details
kubectl get endpoints port-test-service -o yaml

# Test direct pod connectivity
POD_IP=$(kubectl get pod -l app=port-test -o jsonpath='{.items[0].status.podIP}')
kubectl run test-client --image=busybox --rm -it --restart=Never -- wget -qO- http://$POD_IP:80
```

**Fix:**

```bash
# Fix target port
kubectl patch service port-test-service -p '{"spec":{"ports":[{"port":80,"targetPort":80}]}}'

# Verify fix
kubectl get endpoints port-test-service
kubectl run test-client --image=busybox --rm -it --restart=Never -- wget -qO- http://port-test-service
```

### Scenario 6: External Traffic Access Issues

**Setup the Problem:**

```bash
# Create NodePort service
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-test-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: external-test
  template:
    metadata:
      labels:
        app: external-test
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: external-test-service
spec:
  selector:
    app: external-test
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
EOF

kubectl wait --for=condition=Available deployment/external-test-deployment --timeout=60s
```

**Troubleshoot:**

```bash
# Check service
kubectl get service external-test-service

# Get node IP
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
echo "Node IP: $NODE_IP"

# Test external access (from within cluster)
kubectl run test-client --image=busybox --rm -it --restart=Never -- wget -qO- http://$NODE_IP:30080

# Check if port is available
kubectl get service external-test-service -o yaml | grep nodePort

# For Minikube, get service URL
if command -v minikube &> /dev/null; then
    minikube service external-test-service --url
fi

# Check node port allocation
kubectl get services --all-namespaces | grep NodePort
```

**Fix:**

```bash
# If port conflict, change nodePort
kubectl patch service external-test-service -p '{"spec":{"ports":[{"port":80,"targetPort":80,"nodePort":30081}]}}'

# For Minikube, start tunnel if needed
if command -v minikube &> /dev/null; then
    echo "For external access on Minikube, run: minikube service external-test-service"
    echo "Or use: minikube tunnel (in separate terminal)"
fi

# Test internal cluster access
kubectl run test-client --image=busybox --rm -it --restart=Never -- wget -qO- http://external-test-service
```

### Complete Testing Script

**Save this as `test-networking.sh`:**

```bash
#!/bin/bash

echo "=== Kubernetes Networking Troubleshooting Scenarios ==="

echo "=== Basic Network Connectivity Test ==="
# Create test pods
kubectl run network-client --image=busybox --command -- sleep 3600
kubectl run network-server --image=nginx:alpine

kubectl wait --for=condition=Ready pod/network-client --timeout=60s
kubectl wait --for=condition=Ready pod/network-server --timeout=60s

# Get server IP
SERVER_IP=$(kubectl get pod network-server -o jsonpath='{.status.podIP}')
echo "Server IP: $SERVER_IP"

echo "Testing pod-to-pod connectivity..."
kubectl exec network-client -- ping -c 3 $SERVER_IP
kubectl exec network-client -- wget -qO- --timeout=5 http://$SERVER_IP

echo ""
echo "=== DNS Resolution Test ==="
# Create service
kubectl expose pod network-server --port=80 --name=test-service

echo "Testing DNS resolution..."
kubectl exec network-client -- nslookup test-service
kubectl exec network-client -- wget -qO- http://test-service

echo ""
echo "=== Network Information ==="
echo "Cluster nodes:"
kubectl get nodes -o wide

echo ""
echo "CNI pods:"
kubectl get pods -n kube-system | grep -E "calico|weave|flannel|cilium|kube-proxy"

echo ""
echo "CoreDNS status:"
kubectl get pods -n kube-system | grep coredns

echo ""
echo "Services:"
kubectl get services

echo ""
echo "=== Cleanup Commands ==="
echo "kubectl delete pod network-client network-server"
echo "kubectl delete service test-service"
```

### Practice Exercise

1. **Set up network monitoring:**
   ```bash
   # Check CNI components
   kubectl get pods -n kube-system
   
   # Monitor network traffic (if tools available)
   kubectl run netshoot --image=nicolaka/netshoot --rm -it --restart=Never
   ```

2. **Test different service types:**
   ```bash
   # ClusterIP (default)
   kubectl expose deployment <deployment> --port=80
   
   # NodePort
   kubectl expose deployment <deployment> --port=80 --type=NodePort
   
   # LoadBalancer (if supported)
   kubectl expose deployment <deployment> --port=80 --type=LoadBalancer
   ```

3. **Practice network policies:**
   ```bash
   # Create deny-all policy
   # Test specific allow rules
   # Understand ingress vs egress policies
   ```

4. **DNS troubleshooting:**
   ```bash
   # Test service discovery
   # Check CoreDNS configuration
   # Verify DNS resolution patterns
   ```

5. **Clean up:**
   ```bash
   kubectl delete all --all
   kubectl delete networkpolicy --all
   kubectl delete namespace development production 2>/dev/null
   ```

### Advanced Scenarios

**Cross-Cluster Networking:**

```bash
# Service mesh connectivity (if using Istio/Linkerd)
kubectl get pods -n istio-system
kubectl get virtualservices
kubectl get destinationrules
```

**Custom CNI Configuration:**

```bash
# Check CNI configuration
cat /etc/cni/net.d/*  # On nodes
kubectl get nodes -o yaml | grep -A 10 podCIDR

# Network policy with complex rules
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: complex-policy
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 3306
EOF
```

**Load Balancer Testing:**

```bash
# Test load balancing behavior
for i in {1..10}; do
  kubectl exec network-client -- wget -qO- http://test-service | grep hostname
done
```

Remember: Networking troubleshooting often requires understanding the underlying network infrastructure. Use tools like `tcpdump`, `netstat`, and `ss` when available to diagnose traffic flow issues.
