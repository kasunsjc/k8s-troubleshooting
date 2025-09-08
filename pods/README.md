# Pod Troubleshooting Guide

Comprehensive guide for troubleshooting common Pod issues in Kubernetes.

## Table of Contents

1. [Common Pod States](#common-pod-states)
2. [Pod Status: Pending](#pod-status-pending)
3. [Pod Status: CrashLoopBackOff](#pod-status-crashloopbackoff)
4. [Pod Status: ImagePullBackOff](#pod-status-imagepullbackoff)
5. [Pod Status: Terminating](#pod-status-terminating)
6. [Pod Status: Running but Not Working](#pod-status-running-but-not-working)
7. [Resource-Related Issues](#resource-related-issues)
8. [Health Check Issues](#health-check-issues)
9. [Quick Reference Commands](#quick-reference-commands)

---

## Common Pod States

Understanding pod states is crucial for effective troubleshooting:

- **Pending**: Pod has been accepted but not scheduled to a node
- **Running**: Pod is bound to a node and all containers are created
- **Succeeded**: All containers terminated successfully
- **Failed**: All containers terminated, at least one failed
- **Unknown**: Pod state could not be obtained

### Quick Diagnosis Commands

```bash
# Check pod status
kubectl get pods -n <namespace>

# Get detailed pod information
kubectl describe pod <pod-name> -n <namespace>

# Check pod logs
kubectl logs <pod-name> -n <namespace>

# Check previous container logs (if crashed)
kubectl logs <pod-name> -n <namespace> --previous
```

---

## Pod Status: Pending

### Symptoms

- Pod remains in `Pending` state indefinitely
- Pod is not scheduled to any node
- Application is not starting

### Troubleshooting Steps

```bash
# Check pod details and events
kubectl describe pod <pod-name> -n <namespace>

# Check node resources and availability
kubectl get nodes
kubectl describe nodes
kubectl top nodes

# Check if there are taints or node selectors
kubectl get nodes --show-labels
kubectl describe node <node-name> | grep Taints

# Check resource requests vs available resources
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Requests"
```

### Common Causes and Solutions

#### 1. Insufficient Resources

**Problem**: Not enough CPU, memory, or other resources on any node.

```bash
# Check resource usage
kubectl top nodes
kubectl describe nodes | grep -A 5 "Allocated resources"
```

**Solutions**:
- Scale down other pods
- Add more nodes to the cluster
- Reduce resource requests in pod spec
- Use resource quotas to manage consumption

#### 2. Node Selector/Affinity Issues

**Problem**: Pod cannot be scheduled due to node selection constraints.

```bash
# Check pod node selector
kubectl get pod <pod-name> -n <namespace> -o yaml | grep nodeSelector -A 5

# Check pod affinity rules
kubectl get pod <pod-name> -n <namespace> -o yaml | grep affinity -A 20
```

**Solutions**:
- Remove or modify node selectors
- Add appropriate labels to nodes
- Adjust affinity/anti-affinity rules

#### 3. Taints and Tolerations

**Problem**: Pod lacks proper tolerations for node taints.

```bash
# Check node taints
kubectl describe node <node-name> | grep Taints

# Check pod tolerations
kubectl get pod <pod-name> -n <namespace> -o yaml | grep tolerations -A 10
```

**Solutions**:
- Add appropriate tolerations to pod spec
- Remove taints from nodes (if appropriate)
- Use DaemonSets for system pods

#### 4. PersistentVolume Issues

**Problem**: Required PV is not available or cannot be bound.

```bash
# Check PVC status
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>

# Check available PVs
kubectl get pv
```

**Solutions**:
- Create required PersistentVolumes
- Check StorageClass configuration
- Verify PVC access modes match PV

---

## Pod Status: CrashLoopBackOff

### Symptoms

- Pod keeps restarting automatically
- Container exits shortly after starting
- Restart count keeps increasing

### Troubleshooting Steps

```bash
# Check pod logs
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous

# Check restart count and reasons
kubectl describe pod <pod-name> -n <namespace>

# Check liveness and readiness probes
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 15 livenessProbe
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 15 readinessProbe
```

### Common Causes and Solutions

#### 1. Application Errors

**Problem**: Application fails to start due to code issues, missing dependencies, or configuration problems.

```bash
# Check application logs for errors
kubectl logs <pod-name> -n <namespace> --tail=50

# Check environment variables
kubectl describe pod <pod-name> -n <namespace> | grep -A 20 "Environment"
```

**Solutions**:
- Fix application code issues
- Ensure all dependencies are available
- Verify configuration files and environment variables
- Check database connectivity

#### 2. Resource Limits

**Problem**: Container is killed due to exceeding resource limits.

```bash
# Check resource limits and usage
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Limits"
kubectl top pod <pod-name> -n <namespace>

# Check for OOMKilled events
kubectl get events -n <namespace> | grep <pod-name> | grep OOMKilled
```

**Solutions**:
- Increase memory limits
- Optimize application memory usage
- Use horizontal pod autoscaling

#### 3. Probe Configuration Issues

**Problem**: Liveness probes are failing and killing the container.

```bash
# Check probe configuration
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Liveness"

# Test probe endpoint manually
kubectl exec -it <pod-name> -n <namespace> -- curl http://localhost:8080/health
```

**Solutions**:
- Adjust probe timing (initialDelaySeconds, periodSeconds)
- Fix probe endpoint or change probe type
- Increase failure threshold

---

## Pod Status: ImagePullBackOff

### Symptoms

- Cannot pull container image
- Pod stuck in `ImagePullBackOff` or `ErrImagePull`
- Image-related error messages in events

### Troubleshooting Steps

```bash
# Check pod events for image pull errors
kubectl describe pod <pod-name> -n <namespace>

# Check image name and tag in pod spec
kubectl get pod <pod-name> -n <namespace> -o yaml | grep image

# Check imagePullSecrets configuration
kubectl get pod <pod-name> -n <namespace> -o yaml | grep imagePullSecrets -A 5
```

### Common Causes and Solutions

#### 1. Incorrect Image Name or Tag

**Problem**: Image name, tag, or registry URL is wrong.

```bash
# Verify image exists in registry
docker pull <image-name>:<tag>

# Check if tag exists
curl -s https://registry.hub.docker.com/v2/repositories/<image-name>/tags/
```

**Solutions**:
- Verify correct image name and tag
- Check if image exists in the specified registry
- Use `latest` tag cautiously in production

#### 2. Authentication Issues

**Problem**: Cannot authenticate with private registry.

```bash
# Check imagePullSecrets
kubectl get secrets -n <namespace>
kubectl describe secret <image-pull-secret> -n <namespace>

# Test registry authentication
kubectl create secret docker-registry myregistrykey \
  --docker-server=<registry-url> \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email>
```

**Solutions**:
- Create proper imagePullSecrets
- Verify registry credentials
- Check service account configuration

#### 3. Network Issues

**Problem**: Nodes cannot reach the image registry.

```bash
# Test connectivity from node
kubectl run test-connectivity --image=busybox --rm -it --restart=Never -- \
  nslookup <registry-hostname>
```

**Solutions**:
- Check network connectivity
- Verify DNS resolution
- Check firewall rules

---

## Pod Status: Terminating

### Symptoms

- Pod stuck in `Terminating` state
- Pod won't delete despite delete commands
- Graceful shutdown is not completing

### Troubleshooting Steps

```bash
# Check if pod has finalizers
kubectl get pod <pod-name> -n <namespace> -o yaml | grep finalizers -A 5

# Check for stuck volumes or other resources
kubectl describe pod <pod-name> -n <namespace>

# Check what's preventing termination
kubectl get events -n <namespace> | grep <pod-name>
```

### Solutions

#### 1. Remove Finalizers

```bash
# Edit pod to remove finalizers (use with caution)
kubectl patch pod <pod-name> -n <namespace> -p '{"metadata":{"finalizers":[]}}'
```

#### 2. Force Delete

```bash
# Force delete pod (last resort)
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force
```

#### 3. Check Dependent Resources

```bash
# Check for PVCs or other dependencies
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>
```

---

## Pod Status: Running but Not Working

### Symptoms

- Pod shows as `Running` but application is not responding
- Health checks may be failing
- Service is not accessible

### Troubleshooting Steps

```bash
# Check readiness probe status
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Ready"

# Test application directly
kubectl exec -it <pod-name> -n <namespace> -- curl http://localhost:8080/health

# Check service endpoints
kubectl get endpoints <service-name> -n <namespace>

# Port forward to test connectivity
kubectl port-forward pod/<pod-name> 8080:8080 -n <namespace>
```

### Common Issues

#### 1. Readiness Probe Failures

```bash
# Check readiness probe configuration
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 15 readinessProbe
```

#### 2. Application Not Listening on Expected Port

```bash
# Check what ports are open in the container
kubectl exec -it <pod-name> -n <namespace> -- netstat -tlnp
```

#### 3. Startup Time Issues

```bash
# Check startup probe if configured
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 15 startupProbe
```

---

## Resource-Related Issues

### Memory Issues

```bash
# Check memory usage and limits
kubectl top pod <pod-name> -n <namespace>
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Limits"

# Check for OOM events
kubectl get events -n <namespace> | grep <pod-name> | grep "OOMKilled\|oom"
```

### CPU Issues

```bash
# Check CPU usage and throttling
kubectl top pod <pod-name> -n <namespace>

# Check for CPU throttling events
kubectl describe pod <pod-name> -n <namespace> | grep -i throttl
```

### Storage Issues

```bash
# Check disk usage in pod
kubectl exec -it <pod-name> -n <namespace> -- df -h

# Check ephemeral storage limits
kubectl describe pod <pod-name> -n <namespace> | grep ephemeral
```

---

## Health Check Issues

### Liveness Probe Problems

```bash
# Check liveness probe configuration
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Liveness"

# Test probe endpoint
kubectl exec -it <pod-name> -n <namespace> -- curl -f http://localhost:8080/health
```

### Readiness Probe Problems

```bash
# Check readiness probe status
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Readiness"

# Check if pod is ready
kubectl get pod <pod-name> -n <namespace> -o wide
```

### Startup Probe Problems

```bash
# Check startup probe configuration
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Startup"
```

---

## Quick Reference Commands

```bash
# Basic pod information
kubectl get pods -n <namespace>
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod-name> -n <namespace>

# Pod logs
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl logs <pod-name> -n <namespace> -f
kubectl logs <pod-name> -c <container-name> -n <namespace>

# Pod debugging
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
kubectl port-forward pod/<pod-name> <local-port>:<pod-port> -n <namespace>

# Resource monitoring
kubectl top pod <pod-name> -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Pod management
kubectl delete pod <pod-name> -n <namespace>
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force

# Configuration inspection
kubectl get pod <pod-name> -n <namespace> -o yaml
kubectl get pod <pod-name> -n <namespace> -o json
```

## Best Practices

1. **Always check logs first**: Use `kubectl logs` to understand what's happening inside the container
2. **Use describe command**: `kubectl describe pod` provides comprehensive information including events
3. **Monitor resource usage**: Use `kubectl top` to check if resource limits are being hit
4. **Test health probes manually**: Execute probe commands directly in the container
5. **Check dependencies**: Ensure required services, ConfigMaps, and Secrets are available
6. **Use proper resource limits**: Set appropriate CPU and memory limits to prevent resource starvation
7. **Implement proper health checks**: Configure liveness, readiness, and startup probes correctly

---

## Hands-On Testing Scenarios

Practice these scenarios in your Minikube or kind cluster to gain practical pod troubleshooting experience.

### Scenario 1: Pod Stuck in Pending State

**Setup the Problem:**

```bash
# Start minikube with limited resources
minikube start --memory=1024 --cpus=1

# Create a pod with impossible resource requirements
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pending-pod-test
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        memory: "4Gi"
        cpu: "2000m"
EOF
```

**Troubleshoot:**

```bash
# Check pod status
kubectl get pods
kubectl describe pod pending-pod-test

# Check node resources
kubectl describe nodes
kubectl top nodes
```

**Fix:**

```bash
# Reduce resource requirements
kubectl delete pod pending-pod-test
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pending-pod-test-fixed
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
EOF
```

### Scenario 2: CrashLoopBackOff Due to Wrong Command

**Setup the Problem:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: crashloop-test
spec:
  containers:
  - name: app
    image: nginx:alpine
    command: ["nonexistent-command"]
EOF
```

**Troubleshoot:**

```bash
# Check pod status
kubectl get pods
kubectl describe pod crashloop-test

# Check logs
kubectl logs crashloop-test
kubectl logs crashloop-test --previous
```

**Fix:**

```bash
# Fix the command
kubectl delete pod crashloop-test
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: crashloop-test-fixed
spec:
  containers:
  - name: app
    image: nginx:alpine
    command: ["nginx", "-g", "daemon off;"]
EOF
```

### Scenario 3: ImagePullBackOff Error

**Setup the Problem:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: imagepull-test
spec:
  containers:
  - name: app
    image: nonexistent-registry.com/fake-image:latest
EOF
```

**Troubleshoot:**

```bash
# Check pod status
kubectl get pods
kubectl describe pod imagepull-test

# Look at events
kubectl get events --sort-by='.lastTimestamp'
```

**Fix:**

```bash
# Use correct image
kubectl delete pod imagepull-test
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: imagepull-test-fixed
spec:
  containers:
  - name: app
    image: nginx:alpine
EOF
```

### Scenario 4: Liveness Probe Failure

**Setup the Problem:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: liveness-test
spec:
  containers:
  - name: app
    image: nginx:alpine
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /nonexistent
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 2
EOF
```

**Troubleshoot:**

```bash
# Wait for pod to start and then fail
kubectl get pods -w

# Check events and logs
kubectl describe pod liveness-test
kubectl logs liveness-test
```

**Fix:**

```bash
# Fix the liveness probe
kubectl delete pod liveness-test
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: liveness-test-fixed
spec:
  containers:
  - name: app
    image: nginx:alpine
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
EOF
```

### Scenario 5: Pod Stuck in Terminating State

**Setup the Problem:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: terminating-test
  finalizers:
    - example.com/test-finalizer
spec:
  containers:
  - name: app
    image: nginx:alpine
EOF

# Wait for pod to be running, then try to delete
kubectl wait --for=condition=Ready pod/terminating-test --timeout=60s
kubectl delete pod terminating-test
```

**Troubleshoot:**

```bash
# Check why pod won't terminate
kubectl get pods
kubectl describe pod terminating-test
```

**Fix:**

```bash
# Remove finalizers
kubectl patch pod terminating-test -p '{"metadata":{"finalizers":[]}}' --type=merge

# Or force delete (as last resort)
# kubectl delete pod terminating-test --grace-period=0 --force
```

### Scenario 6: Resource Limit Issues (OOMKilled)

**Setup the Problem:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: memory-limit-test
spec:
  containers:
  - name: app
    image: alpine:latest
    command: ["sh", "-c"]
    args: ["dd if=/dev/zero of=/tmp/memory bs=1M count=200; sleep 3600"]
    resources:
      limits:
        memory: "128Mi"
      requests:
        memory: "64Mi"
EOF
```

**Troubleshoot:**

```bash
# Wait for OOMKilled
kubectl get pods -w

# Check what happened
kubectl describe pod memory-limit-test
kubectl get events | grep memory-limit-test
```

**Fix:**

```bash
# Increase memory limit or reduce memory usage
kubectl delete pod memory-limit-test
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: memory-limit-test-fixed
spec:
  containers:
  - name: app
    image: alpine:latest
    command: ["sh", "-c", "sleep 3600"]
    resources:
      limits:
        memory: "128Mi"
      requests:
        memory: "64Mi"
EOF
```

### Scenario 7: Configuration Issues

**Setup the Problem:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-test
spec:
  containers:
  - name: app
    image: nginx:alpine
    env:
    - name: REQUIRED_VAR
      valueFrom:
        configMapKeyRef:
          name: nonexistent-configmap
          key: somekey
EOF
```

**Troubleshoot:**

```bash
# Check pod status
kubectl get pods
kubectl describe pod config-test
```

**Fix:**

```bash
# Create the required ConfigMap first
kubectl create configmap test-config --from-literal=somekey=somevalue

# Or remove the environment variable requirement
kubectl delete pod config-test
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-test-fixed
spec:
  containers:
  - name: app
    image: nginx:alpine
EOF
```

### Complete Testing Script

**Save this as `test-pods.sh`:**

```bash
#!/bin/bash

echo "=== Kubernetes Pod Troubleshooting Scenarios ==="
echo "Make sure you have minikube or kind cluster running!"

echo "Scenario 1: Creating a pod with resource issues..."
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: resource-issue-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        memory: "10Gi"
        cpu: "8000m"
EOF

echo "Scenario 2: Creating a pod with image issues..."
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: image-issue-pod
spec:
  containers:
  - name: app
    image: fake-registry.com/nonexistent:latest
EOF

echo "Scenario 3: Creating a pod with crash issues..."
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: crash-pod
spec:
  containers:
  - name: app
    image: alpine:latest
    command: ["fake-command"]
EOF

echo ""
echo "Now troubleshoot these pods:"
echo "1. kubectl get pods"
echo "2. kubectl describe pod <pod-name>"
echo "3. kubectl logs <pod-name>"
echo ""
echo "Cleanup: kubectl delete pods --all"
```

### Practice Exercise

1. **Set up your environment:**
   ```bash
   # Using minikube
   minikube start --memory=2048 --cpus=2
   
   # Or using kind
   kind create cluster
   ```

2. **Run through each scenario** systematically:
   - Create the problematic pod
   - Use troubleshooting commands to identify the issue
   - Apply the fix
   - Verify the solution works

3. **Try variations:**
   - Change resource values
   - Use different images
   - Modify probe configurations
   - Add different environment variables

4. **Clean up:**
   ```bash
   kubectl delete pods --all
   minikube delete  # or kind delete cluster
   ```

### Advanced Scenarios

**Multi-container pod issues:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-test
spec:
  containers:
  - name: main-app
    image: nginx:alpine
  - name: sidecar
    image: busybox:latest
    command: ["nonexistent-command"]
EOF
```

**Init container problems:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-container-test
spec:
  initContainers:
  - name: init
    image: busybox:latest
    command: ["sh", "-c", "exit 1"]
  containers:
  - name: main
    image: nginx:alpine
EOF
```

Remember: Pod troubleshooting requires understanding the entire container lifecycle, from image pulling to application startup and health monitoring.
