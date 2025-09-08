# Node Troubleshooting Guide

Guide for troubleshooting Kubernetes Node issues and cluster health.

## Common Node Issues

### Node Not Ready

```bash
# Check node status
kubectl get nodes
kubectl describe node <node-name>

# Check node conditions
kubectl get node <node-name> -o yaml | grep -A 20 conditions

# Check kubelet status (on the node)
sudo systemctl status kubelet
sudo journalctl -u kubelet -f

# Check node resources
kubectl top node <node-name>
df -h  # Check disk space
free -h  # Check memory
```

### Node Resource Exhaustion

```bash
# Check resource usage
kubectl top nodes
kubectl describe node <node-name> | grep -A 10 "Allocated resources"

# Check for evicted pods
kubectl get pods --all-namespaces | grep Evicted

# Clean up completed pods
kubectl delete pods --all-namespaces --field-selector=status.phase=Succeeded
kubectl delete pods --all-namespaces --field-selector=status.phase=Failed
```

### Pod Scheduling Issues

```bash
# Check node taints
kubectl describe node <node-name> | grep Taints

# Check node labels
kubectl get node <node-name> --show-labels

# Check cordoned nodes
kubectl get nodes | grep SchedulingDisabled

# Uncordon node if needed
kubectl uncordon <node-name>
```

## Quick Commands

```bash
# Node management
kubectl get nodes
kubectl describe node <node-name>
kubectl cordon <node-name>
kubectl uncordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Resource monitoring
kubectl top nodes
kubectl describe node <node-name> | grep -A 5 "Allocated resources"

# Node troubleshooting
kubectl get events --field-selector involvedObject.kind=Node
kubectl get pods --all-namespaces -o wide | grep <node-name>
```

---

## Hands-On Testing Scenarios

Practice these scenarios in your Minikube or kind cluster to gain practical Node troubleshooting experience.

### Scenario 1: Node Resource Exhaustion

**Setup the Problem:**

```bash
# Check current node resources
kubectl top nodes
kubectl describe nodes

# Create memory-hungry deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-hog
spec:
  replicas: 5
  selector:
    matchLabels:
      app: memory-hog
  template:
    metadata:
      labels:
        app: memory-hog
    spec:
      containers:
      - name: memory-consumer
        image: progrium/stress
        args: ["--vm", "1", "--vm-bytes", "512M", "--vm-hang", "1"]
        resources:
          requests:
            memory: "256Mi"
          limits:
            memory: "512Mi"
EOF
```

**Troubleshoot:**

```bash
# Monitor node resource usage
kubectl top nodes
kubectl top pods

# Check if pods are pending
kubectl get pods -o wide
kubectl describe pods -l app=memory-hog

# Check node capacity
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check for resource pressure
kubectl get nodes -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].type,REASON:.status.conditions[-1].reason
```

**Fix:**

```bash
# Scale down the deployment
kubectl scale deployment memory-hog --replicas=2

# Or add resource requests/limits
kubectl patch deployment memory-hog -p '{"spec":{"template":{"spec":{"containers":[{"name":"memory-consumer","resources":{"requests":{"memory":"100Mi"},"limits":{"memory":"256Mi"}}}]}}}}'

# Verify fix
kubectl get pods -l app=memory-hog
kubectl top nodes
```

### Scenario 2: Node Not Ready (Simulated Kubelet Issues)

**Setup the Problem (Minikube/kind specific):**

```bash
# For Minikube - stop kubelet service
minikube ssh
sudo systemctl stop kubelet
exit

# For kind - we'll simulate by creating a tainted node scenario
kubectl taint nodes --all node.kubernetes.io/unreachable:NoSchedule
```

**Troubleshoot:**

```bash
# Check node status
kubectl get nodes
kubectl describe nodes

# Check node conditions
kubectl get nodes -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[4].type,REASON:.status.conditions[4].reason

# Check if pods are being evicted
kubectl get pods --all-namespaces -o wide
```

**Fix:**

```bash
# For Minikube - restart kubelet
minikube ssh
sudo systemctl start kubelet
exit

# For kind - remove taints
kubectl taint nodes --all node.kubernetes.io/unreachable:NoSchedule-

# Verify nodes are ready
kubectl get nodes
```

### Scenario 3: Disk Pressure Simulation

**Setup the Problem:**

```bash
# Check current disk usage
kubectl describe nodes | grep -A 10 "System Info"

# Create pods that consume disk space
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: disk-filler-1
spec:
  containers:
  - name: disk-consumer
    image: busybox
    command:
    - sh
    - -c
    - |
      df -h /tmp
      dd if=/dev/zero of=/tmp/bigfile1 bs=1M count=100
      sleep 3600
    volumeMounts:
    - name: host-tmp
      mountPath: /tmp
  volumes:
  - name: host-tmp
    hostPath:
      path: /tmp
  nodeSelector:
    kubernetes.io/os: linux
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: disk-filler-2
spec:
  containers:
  - name: disk-consumer
    image: busybox
    command:
    - sh
    - -c
    - |
      dd if=/dev/zero of=/tmp/bigfile2 bs=1M count=100
      sleep 3600
    volumeMounts:
    - name: host-tmp
      mountPath: /tmp
  volumes:
  - name: host-tmp
    hostPath:
      path: /tmp
EOF
```

**Troubleshoot:**

```bash
# Check node conditions for disk pressure
kubectl describe nodes | grep -A 10 "Conditions"

# Check disk usage
kubectl exec disk-filler-1 -- df -h /tmp

# Look for DiskPressure condition
kubectl get nodes -o custom-columns=NAME:.metadata.name,DISK-PRESSURE:.status.conditions[1].status
```

**Fix:**

```bash
# Clean up disk space
kubectl delete pod disk-filler-1 disk-filler-2

# For more thorough cleanup (Minikube)
minikube ssh
sudo rm -f /tmp/bigfile*
exit
```

### Scenario 4: Node Scheduling Issues

**Setup the Problem:**

```bash
# Taint all nodes to prevent scheduling
kubectl taint nodes --all example.com/no-schedule:NoSchedule

# Try to create a pod without toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
EOF
```

**Troubleshoot:**

```bash
# Check pod status
kubectl get pods
kubectl describe pod no-toleration-pod

# Check node taints
kubectl describe nodes | grep -A 5 "Taints"

# Check scheduling events
kubectl get events --sort-by=.metadata.creationTimestamp
```

**Fix:**

```bash
# Add toleration to pod
kubectl delete pod no-toleration-pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: toleration-pod
spec:
  tolerations:
  - key: "example.com/no-schedule"
    operator: "Exists"
    effect: "NoSchedule"
  containers:
  - name: app
    image: nginx:alpine
EOF

# Or remove taint from nodes
kubectl taint nodes --all example.com/no-schedule:NoSchedule-
```

### Scenario 5: Node Labeling and Affinity Issues

**Setup the Problem:**

```bash
# Remove standard labels from nodes (if possible)
kubectl label nodes --all kubernetes.io/os-

# Create pod with node affinity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: affinity-test-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
  containers:
  - name: app
    image: nginx:alpine
EOF
```

**Troubleshoot:**

```bash
# Check pod scheduling
kubectl get pods
kubectl describe pod affinity-test-pod

# Check node labels
kubectl get nodes --show-labels

# Check why pod is not scheduled
kubectl get events --field-selector involvedObject.name=affinity-test-pod
```

**Fix:**

```bash
# Add required label back to nodes
kubectl label nodes --all kubernetes.io/os=linux

# Verify pod gets scheduled
kubectl get pods -w
```

### Scenario 6: Network Connectivity Issues

**Setup the Problem:**

```bash
# Create a pod to test network connectivity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: network-test-pod
spec:
  containers:
  - name: network-tools
    image: nicolaka/netshoot
    command: ["sleep", "3600"]
EOF

kubectl wait --for=condition=Ready pod/network-test-pod --timeout=60s
```

**Troubleshoot:**

```bash
# Test DNS resolution
kubectl exec network-test-pod -- nslookup kubernetes.default.svc.cluster.local

# Test external connectivity
kubectl exec network-test-pod -- ping -c 3 8.8.8.8

# Test internal cluster networking
kubectl exec network-test-pod -- wget -qO- http://kubernetes.default.svc.cluster.local

# Check node network configuration
kubectl describe nodes | grep -A 10 "Addresses"

# Check CNI status (varies by setup)
kubectl get pods -n kube-system | grep -E "(calico|flannel|weave|cilium)"
```

**Fix:**

```bash
# Restart CNI pods if needed (example for calico)
kubectl delete pods -n kube-system -l k8s-app=calico-node

# For Minikube, restart cluster if network issues persist
# minikube delete && minikube start
```

### Complete Testing Script

**Save this as `test-nodes.sh`:**

```bash
#!/bin/bash

echo "=== Kubernetes Node Troubleshooting Scenarios ==="

# Function to check cluster type
check_cluster_type() {
    if kubectl config current-context | grep -q minikube; then
        echo "Detected Minikube cluster"
        CLUSTER_TYPE="minikube"
    elif kubectl config current-context | grep -q kind; then
        echo "Detected kind cluster"
        CLUSTER_TYPE="kind"
    else
        echo "Unknown cluster type"
        CLUSTER_TYPE="unknown"
    fi
}

check_cluster_type

echo ""
echo "Current cluster nodes:"
kubectl get nodes -o wide

echo ""
echo "Node resource usage:"
kubectl top nodes 2>/dev/null || echo "Metrics server not available"

echo ""
echo "=== Scenario 1: Resource Exhaustion Test ==="
echo "Creating memory-hungry pods..."
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: resource-test
  template:
    metadata:
      labels:
        app: resource-test
    spec:
      containers:
      - name: cpu-burner
        image: busybox
        command: ["sh", "-c", "while true; do echo 'CPU burning...'; done"]
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
EOF

echo "Monitor with: kubectl top pods -l app=resource-test"
echo ""

echo "=== Scenario 2: Node Scheduling Test ==="
echo "Adding taint to prevent scheduling..."
kubectl taint nodes --all test-taint:NoSchedule 2>/dev/null || echo "Taints may already exist"

echo "Creating pod without toleration..."
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: scheduling-test
spec:
  containers:
  - name: app
    image: nginx:alpine
EOF

echo "Check scheduling: kubectl describe pod scheduling-test"
echo "Remove taint: kubectl taint nodes --all test-taint:NoSchedule-"
echo ""

echo "=== Cleanup Commands ==="
echo "kubectl delete deployment resource-test"
echo "kubectl delete pod scheduling-test"
echo "kubectl taint nodes --all test-taint:NoSchedule- 2>/dev/null"
```

### Practice Exercise

1. **Set up monitoring:**
   ```bash
   # Install metrics server if not available
   kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
   
   # For Minikube, enable metrics server
   minikube addons enable metrics-server
   ```

2. **Monitor node health:**
   ```bash
   # Watch node status
   kubectl get nodes -w
   
   # Monitor resource usage
   watch kubectl top nodes
   
   # Check system pods
   kubectl get pods -n kube-system
   ```

3. **Test different scenarios:**
   - Resource pressure scenarios
   - Node labeling and scheduling
   - Network connectivity tests
   - Taint and toleration behavior

4. **Learn cluster-specific commands:**
   ```bash
   # Minikube specific
   minikube status
   minikube logs
   minikube ssh
   
   # kind specific
   kind get clusters
   docker exec -it <kind-node> bash
   ```

5. **Clean up:**
   ```bash
   kubectl delete all --all
   kubectl taint nodes --all test-taint:NoSchedule- 2>/dev/null
   ```

### Advanced Scenarios

**Node Drain and Maintenance:**

```bash
# Safely drain a node
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Test pod rescheduling
kubectl get pods -o wide -w

# Uncordon node
kubectl uncordon <node-name>
```

**Node Pressure Simulation:**

```bash
# Create pods that trigger different pressure types
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: memory-pressure-test
spec:
  containers:
  - name: memory-consumer
    image: progrium/stress
    args: ["--vm", "1", "--vm-bytes", "1G"]
    resources:
      limits:
        memory: "1Gi"
EOF
```

Remember: Understanding node behavior is crucial for maintaining a healthy Kubernetes cluster. Practice these scenarios to become familiar with common node issues and their resolutions.
