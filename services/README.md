# Service Troubleshooting Guide

Comprehensive guide for troubleshooting Kubernetes Service connectivity and routing issues.

## Table of Contents

1. [Service Types and Common Issues](#service-types-and-common-issues)
2. [Service Not Accessible](#service-not-accessible)
3. [LoadBalancer Issues](#loadbalancer-issues)
4. [NodePort Issues](#nodeport-issues)
5. [ClusterIP Issues](#clusterip-issues)
6. [Endpoint Problems](#endpoint-problems)
7. [DNS and Service Discovery](#dns-and-service-discovery)
8. [Network Policy Impact](#network-policy-impact)
9. [Quick Reference Commands](#quick-reference-commands)

---

## Service Types and Common Issues

### Service Types Overview

- **ClusterIP**: Internal cluster communication (default)
- **NodePort**: Exposes service on each node's IP at a static port
- **LoadBalancer**: External load balancer (cloud provider dependent)
- **ExternalName**: Maps service to DNS name

### Quick Diagnosis Commands

```bash
# Check service status
kubectl get svc -n <namespace>
kubectl get svc -n <namespace> -o wide

# Get detailed service information
kubectl describe svc <service-name> -n <namespace>

# Check service endpoints
kubectl get endpoints <service-name> -n <namespace>
kubectl describe endpoints <service-name> -n <namespace>
```

---

## Service Not Accessible

### Diagnosis Steps

#### 1. Check Service Configuration

```bash
# Verify service exists and has correct configuration
kubectl get svc <service-name> -n <namespace> -o yaml

# Check service selector
kubectl get svc <service-name> -n <namespace> -o yaml | grep -A 5 selector

# Check service ports
kubectl get svc <service-name> -n <namespace> -o yaml | grep -A 10 ports
```

#### 2. Verify Pod Labels Match Service Selector

```bash
# Check pod labels
kubectl get pods -n <namespace> --show-labels

# Check if any pods match the service selector
kubectl get pods -n <namespace> -l <selector-key>=<selector-value>

# Compare service selector with pod labels
kubectl get svc <service-name> -n <namespace> -o yaml | grep -A 5 selector
```

#### 3. Check Endpoints

```bash
# Verify service has endpoints
kubectl get endpoints <service-name> -n <namespace>

# If no endpoints, check why pods aren't ready
kubectl describe endpoints <service-name> -n <namespace>
```

### Common Causes and Solutions

#### 1. Selector Mismatch

**Problem**: Service selector doesn't match pod labels.

```bash
# Check service selector
kubectl get svc <service-name> -n <namespace> -o jsonpath='{.spec.selector}'

# Check pod labels
kubectl get pods -n <namespace> -o jsonpath='{.items[*].metadata.labels}'
```

**Solutions**:

- Update service selector to match pod labels
- Update pod labels to match service selector
- Ensure consistent labeling strategy

#### 2. No Ready Pods

**Problem**: All target pods are not in ready state.

```bash
# Check pod readiness
kubectl get pods -n <namespace> -l <selector>

# Check why pods aren't ready
kubectl describe pod <pod-name> -n <namespace>
```

**Solutions**:

- Fix pod readiness probe issues
- Resolve application startup problems
- Check resource constraints

#### 3. Port Configuration Issues

**Problem**: Service ports don't match container ports.

```bash
# Check service ports
kubectl get svc <service-name> -n <namespace> -o yaml | grep -A 5 ports

# Check container ports
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 5 ports
```

**Solutions**:

- Update service port configuration
- Verify targetPort matches container port
- Check protocol (TCP/UDP) matches

#### 4. Network Policy Blocking Traffic

**Problem**: Network policies prevent communication.

```bash
# Check network policies
kubectl get networkpolicies -n <namespace>
kubectl describe networkpolicy <policy-name> -n <namespace>

# Check if policies affect the service
kubectl get pods -n <namespace> -l <selector> -o wide
```

**Solutions**:

- Review and update network policies
- Add necessary ingress/egress rules
- Test without network policies temporarily

---

## LoadBalancer Issues

### External IP Not Assigned

#### Diagnosis Steps

```bash
# Check LoadBalancer service status
kubectl get svc <service-name> -n <namespace> -o wide

# Check service events
kubectl describe svc <service-name> -n <namespace>

# Check cloud controller manager logs
kubectl logs -n kube-system -l component=cloud-controller-manager
```

#### Common Issues and Solutions

1. **Cloud Provider Integration**:

```bash
# Verify cloud provider is configured
kubectl get nodes -o yaml | grep -A 5 providerID

# Check cloud controller manager
kubectl get pods -n kube-system | grep cloud-controller
```

2. **Service Annotations**:

```bash
# Check for required cloud provider annotations
kubectl get svc <service-name> -n <namespace> -o yaml | grep annotations -A 10
```

3. **Quota Limits**:

```bash
# Check if you've reached LoadBalancer quota
# This is cloud provider specific
```

### LoadBalancer Traffic Not Reaching Pods

```bash
# Test external connectivity
curl -v http://<external-ip>:<port>

# Check if service endpoints are healthy
kubectl get endpoints <service-name> -n <namespace>

# Test internal connectivity
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -qO- <service-name>:<port>
```

---

## NodePort Issues

### Port Not Accessible from Outside

#### Diagnosis Steps

```bash
# Check NodePort service configuration
kubectl get svc <service-name> -n <namespace> -o yaml | grep -A 10 ports

# Get node IPs
kubectl get nodes -o wide

# Check if port is in valid NodePort range (30000-32767)
kubectl describe svc <service-name> -n <namespace>
```

#### Common Issues

1. **Firewall Rules**:

```bash
# Check if NodePort is accessible from outside
telnet <node-ip> <nodeport>

# Check node firewall settings (on the node)
sudo iptables -L -n | grep <nodeport>
```

2. **Node Network Configuration**:

```bash
# Check if nodes are reachable
ping <node-ip>

# Check kube-proxy status
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system <kube-proxy-pod>
```

---

## ClusterIP Issues

### Internal Service Discovery Problems

#### Diagnosis Steps

```bash
# Test service resolution from within cluster
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup <service-name>.<namespace>.svc.cluster.local

# Test service connectivity
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -qO- <service-name>.<namespace>:<port>

# Check CoreDNS
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system <coredns-pod>
```

#### Common Issues

1. **DNS Resolution**:

```bash
# Check DNS configuration in pods
kubectl exec -it <pod-name> -n <namespace> -- cat /etc/resolv.conf

# Test different service name formats
kubectl exec -it <pod-name> -n <namespace> -- nslookup <service-name>
kubectl exec -it <pod-name> -n <namespace> -- nslookup <service-name>.<namespace>
kubectl exec -it <pod-name> -n <namespace> -- nslookup <service-name>.<namespace>.svc.cluster.local
```

2. **Service Mesh Interference**:

```bash
# Check if service mesh is affecting traffic
kubectl get pods -n <namespace> -o yaml | grep -A 5 sidecar
```

---

## Endpoint Problems

### No Endpoints Available

#### Diagnosis Steps

```bash
# Check if service has any endpoints
kubectl get endpoints <service-name> -n <namespace>

# If no endpoints, check pod status
kubectl get pods -n <namespace> -l <service-selector>

# Check pod readiness
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Ready"
```

### Endpoints Not Updating

```bash
# Check endpoint controller
kubectl get pods -n kube-system | grep endpoint
kubectl logs -n kube-system <endpoint-controller-pod>

# Force endpoint refresh by restarting service
kubectl delete svc <service-name> -n <namespace>
kubectl apply -f <service-manifest>
```

---

## DNS and Service Discovery

### Service DNS Not Resolving

#### Diagnosis Steps

```bash
# Test DNS resolution
kubectl exec -it <pod-name> -n <namespace> -- nslookup kubernetes.default.svc.cluster.local

# Check CoreDNS configuration
kubectl get configmap coredns -n kube-system -o yaml

# Check CoreDNS pods
kubectl get pods -n kube-system | grep coredns
kubectl describe pod <coredns-pod> -n kube-system
```

#### Common DNS Issues

1. **CoreDNS Not Running**:

```bash
# Check CoreDNS deployment
kubectl get deployment coredns -n kube-system
kubectl describe deployment coredns -n kube-system
```

2. **DNS Policy Issues**:

```bash
# Check pod DNS policy
kubectl get pod <pod-name> -n <namespace> -o yaml | grep dnsPolicy
```

3. **Custom DNS Configuration**:

```bash
# Check for custom DNS settings
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 10 dnsConfig
```

---

## Network Policy Impact

### Testing Network Policy Effects

```bash
# List network policies affecting namespace
kubectl get networkpolicies -n <namespace>

# Check specific policy rules
kubectl describe networkpolicy <policy-name> -n <namespace>

# Test connectivity with and without policies
kubectl label namespace <namespace> name=<namespace>
# Apply/remove policies and test
```

### Debugging Network Policy Issues

```bash
# Check if pods are selected by network policy
kubectl get pods -n <namespace> --show-labels

# Test connectivity between pods
kubectl exec -it <source-pod> -n <namespace> -- nc -zv <target-pod-ip> <port>

# Check policy logs (if available)
# This depends on your CNI plugin
```

---

## Service Testing and Validation

### Manual Service Testing

```bash
# Create test pod for connectivity testing
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-connectivity
  namespace: <namespace>
spec:
  containers:
  - name: test
    image: busybox
    command: ['sleep', '3600']
EOF

# Test service connectivity
kubectl exec -it test-connectivity -n <namespace> -- wget -qO- <service-name>:<port>
kubectl exec -it test-connectivity -n <namespace> -- nc -zv <service-name> <port>
```

### Service Performance Testing

```bash
# Test service response time
kubectl exec -it test-connectivity -n <namespace> -- time wget -qO- <service-name>:<port>

# Test multiple connections
kubectl exec -it test-connectivity -n <namespace> -- \
  sh -c 'for i in $(seq 1 10); do wget -qO- <service-name>:<port> > /dev/null && echo "Success $i" || echo "Failed $i"; done'
```

---

## Quick Reference Commands

```bash
# Service management
kubectl create service clusterip <name> --tcp=<port>:<target-port> -n <namespace>
kubectl expose deployment <deployment-name> --port=<port> --target-port=<target-port> -n <namespace>
kubectl get svc -n <namespace>
kubectl describe svc <service-name> -n <namespace>
kubectl delete svc <service-name> -n <namespace>

# Endpoint inspection
kubectl get endpoints -n <namespace>
kubectl describe endpoints <service-name> -n <namespace>

# Service testing
kubectl run test-pod --image=busybox --rm -it --restart=Never -- sh
# Inside pod: wget -qO- <service-name>:<port>
# Inside pod: nslookup <service-name>.<namespace>.svc.cluster.local

# Port forwarding for testing
kubectl port-forward svc/<service-name> <local-port>:<service-port> -n <namespace>

# Service configuration
kubectl get svc <service-name> -n <namespace> -o yaml
kubectl patch svc <service-name> -p '{"spec":{"selector":{"app":"new-app"}}}' -n <namespace>

# DNS testing
kubectl exec -it <pod-name> -n <namespace> -- nslookup <service-name>
kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default.svc.cluster.local
```

## Best Practices

1. **Consistent Labeling**: Use consistent labels for pods and service selectors
2. **Health Checks**: Implement proper readiness probes for service endpoints
3. **Service Naming**: Use descriptive service names following DNS conventions
4. **Port Management**: Document and standardize port usage across services
5. **Network Policies**: Design network policies with service communication in mind
6. **Monitoring**: Monitor service endpoints and response times
7. **Testing**: Regularly test service connectivity and DNS resolution
8. **Documentation**: Document service dependencies and communication patterns

## Service Patterns

### Headless Services

```bash
# Create headless service (clusterIP: None)
kubectl create service clusterip <name> --clusterip=None --tcp=<port> -n <namespace>

# Test headless service DNS resolution
kubectl exec -it <pod-name> -n <namespace> -- nslookup <service-name>.<namespace>.svc.cluster.local
```

### Multi-Port Services

```bash
# Check multi-port service configuration
kubectl get svc <service-name> -n <namespace> -o yaml | grep -A 15 ports

# Test specific ports
kubectl exec -it <pod-name> -n <namespace> -- nc -zv <service-name> <port1>
kubectl exec -it <pod-name> -n <namespace> -- nc -zv <service-name> <port2>
```

### External Services

```bash
# Create ExternalName service
kubectl create service externalname <name> --external-name=<external-hostname> -n <namespace>

# Test external service resolution
kubectl exec -it <pod-name> -n <namespace> -- nslookup <service-name>.<namespace>.svc.cluster.local
```

---

## Hands-On Testing Scenarios

Practice these scenarios in your Minikube or kind cluster to gain practical service troubleshooting experience.

### Scenario 1: Service Selector Mismatch

**Setup the Problem:**

```bash
# Create a deployment with specific labels
kubectl create deployment web-app --image=nginx:alpine
kubectl label deployment web-app version=v1

# Create a service with wrong selector
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: wrong-label  # This doesn't match the deployment
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
```

**Troubleshoot:**

```bash
# Check service and endpoints
kubectl get svc web-service
kubectl get endpoints web-service
kubectl describe svc web-service

# Check pod labels
kubectl get pods --show-labels
kubectl get deployment web-app -o yaml | grep -A 5 labels
```

**Fix:**

```bash
# Update service selector to match pod labels
kubectl patch svc web-service -p '{"spec":{"selector":{"app":"web-app"}}}'

# Verify fix
kubectl get endpoints web-service
```

### Scenario 2: Port Configuration Issues

**Setup the Problem:**

```bash
# Create deployment with custom port
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-port-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: custom-port-app
  template:
    metadata:
      labels:
        app: custom-port-app
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: wrong-port-service
spec:
  selector:
    app: custom-port-app
  ports:
  - port: 80
    targetPort: 8080  # Wrong target port
EOF
```

**Troubleshoot:**

```bash
# Test service connectivity
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -qO- wrong-port-service:80

# Check service configuration
kubectl describe svc wrong-port-service
kubectl describe pod <pod-name> | grep -A 5 "Ports"
```

**Fix:**

```bash
# Fix the target port
kubectl patch svc wrong-port-service -p '{"spec":{"ports":[{"port":80,"targetPort":80}]}}'

# Test again
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -qO- wrong-port-service:80
```

### Scenario 3: No Ready Endpoints

**Setup the Problem:**

```bash
# Create deployment with failing readiness probe
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unready-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: unready-app
  template:
    metadata:
      labels:
        app: unready-app
    spec:
      containers:
      - name: app
        image: nginx:alpine
        readinessProbe:
          httpGet:
            path: /health  # Non-existent endpoint
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: unready-service
spec:
  selector:
    app: unready-app
  ports:
  - port: 80
    targetPort: 80
EOF
```

**Troubleshoot:**

```bash
# Check service endpoints
kubectl get endpoints unready-service
kubectl describe endpoints unready-service

# Check pod readiness
kubectl get pods
kubectl describe pod <pod-name>
```

**Fix:**

```bash
# Fix the readiness probe
kubectl patch deployment unready-app -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","readinessProbe":{"httpGet":{"path":"/","port":80}}}]}}}}'

# Wait and check endpoints
kubectl get endpoints unready-service
```

### Scenario 4: NodePort Accessibility Issues

**Setup the Problem:**

```bash
# Create NodePort service
kubectl create deployment nodeport-app --image=nginx:alpine
kubectl expose deployment nodeport-app --type=NodePort --port=80

# Get the assigned NodePort
NODEPORT=$(kubectl get svc nodeport-app -o jsonpath='{.spec.ports[0].nodePort}')
echo "NodePort: $NODEPORT"

# Try to access from outside (this might fail in some environments)
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
echo "Node IP: $NODE_IP"
```

**Troubleshoot:**

```bash
# Check service configuration
kubectl get svc nodeport-app
kubectl describe svc nodeport-app

# Check if port is accessible from within cluster
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nc -zv $NODE_IP $NODEPORT

# For minikube, use minikube service command
minikube service nodeport-app --url
```

### Scenario 5: LoadBalancer Service Issues

**Setup the Problem:**

```bash
# Create LoadBalancer service (will stay pending in local clusters)
kubectl create deployment lb-app --image=nginx:alpine
kubectl expose deployment lb-app --type=LoadBalancer --port=80

# Check service status
kubectl get svc lb-app
```

**Troubleshoot:**

```bash
# Check why external IP is pending
kubectl describe svc lb-app

# Check events
kubectl get events | grep lb-app

# In minikube, use tunnel to simulate LoadBalancer
# minikube tunnel  # Run in separate terminal
```

**Fix (for local testing):**

```bash
# Change to NodePort for local testing
kubectl patch svc lb-app -p '{"spec":{"type":"NodePort"}}'

# Or use port-forward
kubectl port-forward svc/lb-app 8080:80
```

### Scenario 6: DNS Resolution Issues

**Setup the Problem:**

```bash
# Create service in different namespace
kubectl create namespace test-ns
kubectl create deployment dns-test --image=nginx:alpine -n test-ns
kubectl expose deployment dns-test --port=80 -n test-ns

# Test DNS resolution from default namespace
kubectl run dns-client --image=busybox --rm -it --restart=Never -- nslookup dns-test
```

**Troubleshoot:**

```bash
# Check CoreDNS
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system <coredns-pod>

# Test different DNS names
kubectl run dns-client --image=busybox --rm -it --restart=Never -- nslookup dns-test.test-ns.svc.cluster.local

# Check DNS configuration in pod
kubectl run dns-client --image=busybox --rm -it --restart=Never -- cat /etc/resolv.conf
```

### Scenario 7: Headless Service Testing

**Setup the Problem:**

```bash
# Create headless service
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: headless-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: headless-app
  template:
    metadata:
      labels:
        app: headless-app
    spec:
      containers:
      - name: app
        image: nginx:alpine
---
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  clusterIP: None  # Headless service
  selector:
    app: headless-app
  ports:
  - port: 80
EOF
```

**Test:**

```bash
# Check DNS resolution for headless service
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup headless-service

# Check individual pod IPs
kubectl get pods -o wide
kubectl get endpoints headless-service
```

### Complete Testing Script

**Save this as `test-services.sh`:**

```bash
#!/bin/bash

echo "=== Kubernetes Service Troubleshooting Scenarios ==="

# Scenario 1: Wrong selector
echo "Creating deployment and service with selector mismatch..."
kubectl create deployment test-app --image=nginx:alpine
kubectl create service clusterip test-service --tcp=80:80 --dry-run=client -o yaml | \
  sed 's/app: test-app/app: wrong-app/' | kubectl apply -f -

echo "Check endpoints: kubectl get endpoints test-service"
echo "Fix: kubectl patch svc test-service -p '{\"spec\":{\"selector\":{\"app\":\"test-app\"}}}'"
echo ""

# Scenario 2: Port issues
echo "Creating service with wrong port..."
kubectl create deployment port-test --image=nginx:alpine
kubectl expose deployment port-test --port=80 --target-port=8080

echo "Test: kubectl run test --image=busybox --rm -it --restart=Never -- wget -qO- port-test:80"
echo "Fix: kubectl patch svc port-test -p '{\"spec\":{\"ports\":[{\"port\":80,\"targetPort\":80}]}}'"
echo ""

echo "Cleanup: kubectl delete deployment,service test-app test-service port-test"
```

### Practice Exercise

1. **Set up your environment:**
   ```bash
   minikube start
   # or
   kind create cluster
   ```

2. **Run each scenario** and practice:
   - Creating the problematic configuration
   - Using diagnostic commands
   - Applying fixes
   - Verifying the solution

3. **Test service communication:**
   ```bash
   # Create test pods for connectivity testing
   kubectl run client --image=busybox --command -- sleep 3600
   kubectl exec -it client -- wget -qO- <service-name>:<port>
   ```

4. **Monitor with tools:**
   ```bash
   # Watch service changes
   kubectl get svc -w
   
   # Monitor endpoints
   kubectl get endpoints -w
   ```

5. **Clean up:**
   ```bash
   kubectl delete all --all
   ```

### Advanced Testing

**Multi-port services:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: multi-port-svc
spec:
  selector:
    app: web-app
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443
  - name: metrics
    port: 9090
    targetPort: 9090
EOF
```

**External services:**

```bash
# Test ExternalName service
kubectl create service externalname external-db --external-name=postgresql.example.com
kubectl run test --image=busybox --rm -it --restart=Never -- nslookup external-db
```

Remember: Service troubleshooting often involves understanding the entire network stack, from DNS resolution to packet routing and firewall rules.
