# Resource Management Troubleshooting Guide

Guide for troubleshooting Kubernetes resource quotas, limits, and resource management issues.

## Common Resource Issues

### Resource Quota Exceeded

```bash
# Check resource quotas
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota -n <namespace>

# Check current resource usage
kubectl top pods -n <namespace>
kubectl describe namespace <namespace>

# Check limit ranges
kubectl get limitrange -n <namespace>
kubectl describe limitrange -n <namespace>
```

### Pod Resource Limits

```bash
# Check pod resource requests and limits
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Limits\|Requests"

# Check if pod is being throttled or OOMKilled
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Last State"
kubectl get events -n <namespace> | grep <pod-name>

# Monitor resource usage
kubectl top pod <pod-name> -n <namespace>
```

### Cluster Resource Monitoring

```bash
# Check overall cluster resources
kubectl top nodes
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check resource pressure
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, conditions: .status.conditions[] | select(.type == "MemoryPressure" or .type == "DiskPressure" or .type == "PIDPressure")}'

# Check resource availability
kubectl describe nodes | grep -E "cpu|memory" | grep -E "Capacity|Allocatable|Allocated"
```

### HPA (Horizontal Pod Autoscaler) Issues

```bash
# Check HPA status
kubectl get hpa -n <namespace>
kubectl describe hpa <hpa-name> -n <namespace>

# Check metrics availability
kubectl top pods -n <namespace>
kubectl get apiservice | grep metrics

# Check HPA events
kubectl get events -n <namespace> | grep HorizontalPodAutoscaler
```

## Quick Commands

```bash
# Resource monitoring
kubectl top nodes
kubectl top pods -n <namespace>
kubectl top pods --all-namespaces --sort-by=cpu
kubectl top pods --all-namespaces --sort-by=memory

# Resource quotas and limits
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota -n <namespace>
kubectl get limitrange -n <namespace>
kubectl describe limitrange -n <namespace>

# Node resource inspection
kubectl describe nodes | grep -A 5 "Allocated resources"
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, capacity: .status.capacity, allocatable: .status.allocatable}'

# Autoscaling
kubectl get hpa -n <namespace>
kubectl describe hpa <hpa-name> -n <namespace>
kubectl autoscale deployment <deployment-name> --min=1 --max=10 --cpu-percent=80 -n <namespace>

# Resource cleanup
kubectl delete pods --all-namespaces --field-selector=status.phase=Succeeded
kubectl delete pods --all-namespaces --field-selector=status.phase=Failed
```

---

## Hands-On Testing Scenarios

Practice these scenarios in your Minikube or kind cluster to gain practical Resource Management troubleshooting experience.

### Scenario 1: Resource Quota Violations

**Setup the Problem:**

```bash
# Create namespace with strict resource quota
kubectl create namespace resource-quota-test

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: strict-quota
  namespace: resource-quota-test
spec:
  hard:
    requests.cpu: "200m"
    requests.memory: "256Mi"
    limits.cpu: "400m"
    limits.memory: "512Mi"
    pods: "3"
    services: "2"
    persistentvolumeclaims: "1"
EOF

# Try to create deployment that exceeds quota
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quota-violation-deployment
  namespace: resource-quota-test
spec:
  replicas: 4  # Exceeds pod quota of 3
  selector:
    matchLabels:
      app: quota-test
  template:
    metadata:
      labels:
        app: quota-test
    spec:
      containers:
      - name: app
        image: nginx:alpine
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
EOF
```

**Troubleshoot:**

```bash
# Check resource quota status
kubectl describe resourcequota strict-quota -n resource-quota-test

# Check deployment status
kubectl get deployment quota-violation-deployment -n resource-quota-test
kubectl describe deployment quota-violation-deployment -n resource-quota-test

# Check events for quota violations
kubectl get events -n resource-quota-test --sort-by=.metadata.creationTimestamp

# Check current resource usage
kubectl top pods -n resource-quota-test
```

**Fix:**

```bash
# Scale down to comply with quota
kubectl scale deployment quota-violation-deployment --replicas=3 -n resource-quota-test

# Verify fix
kubectl get pods -n resource-quota-test
kubectl describe resourcequota strict-quota -n resource-quota-test
```

### Scenario 2: LimitRange Constraint Violations

**Setup the Problem:**

```bash
# Create namespace with limit ranges
kubectl create namespace limitrange-test

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: strict-limits
  namespace: limitrange-test
spec:
  limits:
  - type: Container
    default:
      cpu: "100m"
      memory: "128Mi"
    defaultRequest:
      cpu: "50m"
      memory: "64Mi"
    min:
      cpu: "10m"
      memory: "32Mi"
    max:
      cpu: "200m"
      memory: "256Mi"
  - type: Pod
    max:
      cpu: "300m"
      memory: "512Mi"
EOF

# Try to create pod that violates limits
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: limit-violating-pod
  namespace: limitrange-test
spec:
  containers:
  - name: cpu-hog
    image: nginx:alpine
    resources:
      requests:
        cpu: "5m"      # Below minimum
        memory: "16Mi"  # Below minimum
      limits:
        cpu: "500m"    # Above maximum
        memory: "1Gi"   # Above maximum
EOF
```

**Troubleshoot:**

```bash
# Check pod creation
kubectl get pods -n limitrange-test
kubectl describe pod limit-violating-pod -n limitrange-test

# Check limit range
kubectl describe limitrange strict-limits -n limitrange-test

# Check events
kubectl get events -n limitrange-test --sort-by=.metadata.creationTimestamp
```

**Fix:**

```bash
# Create compliant pod
kubectl delete pod limit-violating-pod -n limitrange-test

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: compliant-pod
  namespace: limitrange-test
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "128Mi"
EOF

# Verify success
kubectl get pods -n limitrange-test
```

### Scenario 3: HPA Not Scaling Due to Missing Metrics

**Setup the Problem:**

```bash
# Create deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-test-deployment
  namespace: resource-quota-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-test
  template:
    metadata:
      labels:
        app: hpa-test
    spec:
      containers:
      - name: app
        image: php:apache
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
EOF

kubectl wait --for=condition=Available deployment/hpa-test-deployment -n resource-quota-test --timeout=60s

# Create HPA
kubectl autoscale deployment hpa-test-deployment --cpu-percent=50 --min=1 --max=5 -n resource-quota-test
```

**Troubleshoot:**

```bash
# Check HPA status
kubectl get hpa -n resource-quota-test
kubectl describe hpa hpa-test-deployment -n resource-quota-test

# Check if metrics server is running
kubectl get pods -n kube-system | grep metrics-server
kubectl get apiservice | grep metrics

# Check metrics availability
kubectl top pods -n resource-quota-test
kubectl top nodes

# Check HPA events
kubectl get events -n resource-quota-test | grep HorizontalPodAutoscaler
```

**Fix:**

```bash
# For Minikube, enable metrics server
if command -v minikube &> /dev/null; then
    minikube addons enable metrics-server
fi

# For kind or other clusters, install metrics server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Wait for metrics server to be ready
kubectl wait --for=condition=Available deployment/metrics-server -n kube-system --timeout=60s

# Check HPA again
kubectl get hpa -n resource-quota-test
```

### Scenario 4: Node Resource Pressure

**Setup the Problem:**

```bash
# Create resource-intensive deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-hog-deployment
spec:
  replicas: 10
  selector:
    matchLabels:
      app: resource-hog
  template:
    metadata:
      labels:
        app: resource-hog
    spec:
      containers:
      - name: cpu-memory-hog
        image: progrium/stress
        args: ["--cpu", "1", "--vm", "1", "--vm-bytes", "128M"]
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "200m"
            memory: "512Mi"
EOF
```

**Troubleshoot:**

```bash
# Check pod scheduling
kubectl get pods -o wide | grep resource-hog

# Check node resource allocation
kubectl describe nodes | grep -A 10 "Allocated resources"

# Check for pending pods
kubectl get pods | grep Pending
kubectl describe pods | grep -A 10 "Events:"

# Monitor resource usage
kubectl top nodes
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory

# Check for resource pressure
kubectl get nodes -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].type,REASON:.status.conditions[-1].reason
```

**Fix:**

```bash
# Scale down deployment
kubectl scale deployment resource-hog-deployment --replicas=3

# Or add node affinity to spread pods
kubectl patch deployment resource-hog-deployment -p '{"spec":{"template":{"spec":{"affinity":{"podAntiAffinity":{"preferredDuringSchedulingIgnoredDuringExecution":[{"weight":100,"podAffinityTerm":{"labelSelector":{"matchExpressions":[{"key":"app","operator":"In","values":["resource-hog"]}]},"topologyKey":"kubernetes.io/hostname"}}]}}}}}}'

# Check resource usage after scaling
kubectl top nodes
kubectl get pods -o wide | grep resource-hog
```

### Scenario 5: VPA (Vertical Pod Autoscaler) Issues

**Setup the Problem:**

```bash
# Note: VPA requires separate installation in most clusters
# Create deployment for VPA testing
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vpa-test-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vpa-test
  template:
    metadata:
      labels:
        app: vpa-test
    spec:
      containers:
      - name: app
        image: nginx:alpine
        resources:
          requests:
            cpu: "10m"     # Intentionally small
            memory: "16Mi" # Intentionally small
          limits:
            cpu: "50m"
            memory: "64Mi"
EOF

# Create VPA configuration (will fail if VPA not installed)
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-test-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vpa-test-deployment
  updatePolicy:
    updateMode: "Auto"
EOF
```

**Troubleshoot:**

```bash
# Check if VPA CRDs exist
kubectl get crd | grep verticalpodautoscaler

# Check VPA controller
kubectl get pods -n kube-system | grep vpa

# Check VPA status
kubectl get vpa
kubectl describe vpa vpa-test-vpa

# Generate load to trigger VPA recommendations
kubectl run load-generator --image=busybox --rm -it --restart=Never -- sh -c "while true; do wget -q -O- http://vpa-test-deployment.default.svc.cluster.local; done"
```

**Fix:**

```bash
# If VPA is not installed, install it (example for demonstration)
echo "VPA installation required. For demo, we'll check resource recommendations manually."

# Monitor actual resource usage
kubectl top pods -l app=vpa-test

# Manually adjust resources based on monitoring
kubectl patch deployment vpa-test-deployment -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","resources":{"requests":{"cpu":"25m","memory":"32Mi"},"limits":{"cpu":"100m","memory":"128Mi"}}}]}}}}'

# Remove VPA if not working
kubectl delete vpa vpa-test-vpa 2>/dev/null || echo "VPA removed or not found"
```

### Scenario 6: Resource Monitoring and Alerting

**Setup the Problem:**

```bash
# Create deployment with varying resource usage
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitoring-test-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: monitoring-test
  template:
    metadata:
      labels:
        app: monitoring-test
    spec:
      containers:
      - name: variable-load
        image: busybox
        command:
        - sh
        - -c
        - |
          while true; do
            echo "Low load period"
            sleep 30
            echo "High load period"
            dd if=/dev/zero of=/dev/null bs=1M count=100 &
            sleep 30
            pkill dd
          done
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
EOF

kubectl wait --for=condition=Available deployment/monitoring-test-deployment --timeout=60s
```

**Troubleshoot:**

```bash
# Monitor resource patterns over time
echo "Monitoring resource usage patterns..."
for i in {1..6}; do
  echo "=== Iteration $i ==="
  kubectl top pods -l app=monitoring-test
  kubectl top nodes
  sleep 15
done

# Check for resource alerts (if monitoring is set up)
kubectl get events --sort-by=.metadata.creationTimestamp | grep -E "(FailedMount|Unhealthy|FailedScheduling)"

# Identify resource bottlenecks
kubectl describe nodes | grep -E "(cpu|memory)" -A 3
```

**Fix:**

```bash
# Set up resource monitoring best practices
echo "Resource monitoring recommendations:"
echo "1. Set appropriate resource requests and limits"
echo "2. Monitor actual vs requested resources"
echo "3. Set up HPA for automatic scaling"
echo "4. Use resource quotas to prevent resource exhaustion"

# Example: Add HPA to handle load spikes
kubectl autoscale deployment monitoring-test-deployment --cpu-percent=70 --min=3 --max=10

# Example: Add resource quotas if not present
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: monitoring-quota
  namespace: default
spec:
  hard:
    requests.cpu: "1000m"
    requests.memory: "2Gi"
    limits.cpu: "2000m"
    limits.memory: "4Gi"
    pods: "20"
EOF
```

### Complete Testing Script

**Save this as `test-resources.sh`:**

```bash
#!/bin/bash

echo "=== Kubernetes Resource Management Troubleshooting Scenarios ==="

echo "=== Checking Cluster Resource Status ==="
echo "Nodes:"
kubectl get nodes -o wide

echo ""
echo "Current resource usage:"
kubectl top nodes 2>/dev/null || echo "Metrics server not available"

echo ""
echo "=== Scenario 1: Basic Resource Requests and Limits ==="
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-test-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "128Mi"
EOF

kubectl wait --for=condition=Ready pod/resource-test-pod --timeout=60s

echo "Pod resource usage:"
kubectl top pod resource-test-pod 2>/dev/null || echo "Metrics not available yet"

echo ""
echo "=== Scenario 2: HPA Setup ==="
kubectl create deployment hpa-demo --image=nginx:alpine
kubectl set resources deployment hpa-demo --requests=cpu=50m,memory=64Mi --limits=cpu=100m,memory=128Mi

echo "Creating HPA..."
kubectl autoscale deployment hpa-demo --cpu-percent=50 --min=1 --max=5

echo "HPA status:"
kubectl get hpa

echo ""
echo "=== Scenario 3: Resource Quota Demo ==="
kubectl create namespace quota-demo 2>/dev/null || echo "quota-demo namespace exists"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: demo-quota
  namespace: quota-demo
spec:
  hard:
    requests.cpu: "200m"
    requests.memory: "256Mi"
    limits.cpu: "400m"
    limits.memory: "512Mi"
    pods: "3"
EOF

echo "Resource quota created. Status:"
kubectl describe resourcequota demo-quota -n quota-demo

echo ""
echo "=== Current Resource Information ==="
echo "All resource quotas:"
kubectl get resourcequota --all-namespaces

echo ""
echo "All limit ranges:"
kubectl get limitrange --all-namespaces

echo ""
echo "All HPAs:"
kubectl get hpa --all-namespaces

echo ""
echo "=== Cleanup Commands ==="
echo "kubectl delete pod resource-test-pod"
echo "kubectl delete deployment hpa-demo"
echo "kubectl delete hpa hpa-demo"
echo "kubectl delete namespace quota-demo"
```

### Practice Exercise

1. **Set up resource monitoring:**
   ```bash
   # Install metrics server if not available
   kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
   
   # For Minikube
   minikube addons enable metrics-server
   ```

2. **Practice resource management:**
   ```bash
   # Set up resource quotas and limits
   # Create deployments with different resource requirements
   # Test HPA scaling behavior
   ```

3. **Monitor resource usage:**
   ```bash
   # Use kubectl top commands
   # Monitor node and pod resource consumption
   # Identify resource bottlenecks
   ```

4. **Test autoscaling:**
   ```bash
   # Set up HPA with different metrics
   # Generate load to trigger scaling
   # Monitor scaling behavior
   ```

5. **Clean up:**
   ```bash
   kubectl delete all --all
   kubectl delete resourcequota --all
   kubectl delete limitrange --all
   kubectl delete hpa --all
   kubectl delete namespace resource-quota-test limitrange-test quota-demo 2>/dev/null
   ```

### Advanced Scenarios

**Custom Metrics HPA:**

```bash
# Example HPA with custom metrics (requires custom metrics API)
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: custom-metrics-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: monitoring-test-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
EOF
```

**Resource Policies:**

```bash
# Pod Disruption Budget for availability during resource pressure
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: monitoring-test-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: monitoring-test
EOF
```

**Node Resource Monitoring:**

```bash
# Check node allocatable vs capacity
kubectl get nodes -o custom-columns=NAME:.metadata.name,CAPACITY-CPU:.status.capacity.cpu,ALLOCATABLE-CPU:.status.allocatable.cpu,CAPACITY-MEMORY:.status.capacity.memory,ALLOCATABLE-MEMORY:.status.allocatable.memory

# Monitor node conditions
kubectl get nodes -o custom-columns=NAME:.metadata.name,READY:.status.conditions[3].status,MEMORY-PRESSURE:.status.conditions[0].status,DISK-PRESSURE:.status.conditions[1].status,PID-PRESSURE:.status.conditions[2].status
```

Remember: Resource management is crucial for cluster stability and performance. Always set appropriate requests and limits, monitor resource usage patterns, and use autoscaling to handle varying workloads efficiently.
