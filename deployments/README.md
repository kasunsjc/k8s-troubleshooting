# Deployment Troubleshooting Guide

Comprehensive guide for troubleshooting Kubernetes Deployment issues.

## Table of Contents

1. [Common Deployment Issues](#common-deployment-issues)
2. [Deployment Not Rolling Out](#deployment-not-rolling-out)
3. [Old Pods Not Terminating](#old-pods-not-terminating)
4. [Failed Rollout](#failed-rollout)
5. [Scaling Issues](#scaling-issues)
6. [Resource Management](#resource-management)
7. [Rollback Procedures](#rollback-procedures)
8. [Quick Reference Commands](#quick-reference-commands)

---

## Common Deployment Issues

### Quick Diagnosis Commands

```bash
# Check deployment status
kubectl get deployments -n <namespace>
kubectl get deployments -n <namespace> -o wide

# Get detailed deployment information
kubectl describe deployment <deployment-name> -n <namespace>

# Check rollout status
kubectl rollout status deployment/<deployment-name> -n <namespace>

# Check deployment history
kubectl rollout history deployment/<deployment-name> -n <namespace>
```

### Understanding Deployment Status

- **Available**: Number of available replicas
- **Ready**: Number of ready replicas
- **Up-to-date**: Number of replicas updated to latest revision
- **Replicas**: Total number of replicas

---

## Deployment Not Rolling Out

### Symptoms and Diagnosis

- Deployment shows old and new replica sets
- Desired replica count not reached
- Rollout appears stuck

### Troubleshooting Steps

```bash
# Check deployment status and events
kubectl describe deployment <deployment-name> -n <namespace>

# Check replica sets
kubectl get rs -n <namespace>
kubectl describe rs <replicaset-name> -n <namespace>

# Check pod status in new replica set
kubectl get pods -n <namespace> -l app=<app-label>

# Check deployment configuration
kubectl get deployment <deployment-name> -n <namespace> -o yaml
```

### Common Causes and Solutions

#### 1. Resource Constraints

**Problem**: Insufficient cluster resources to create new pods.

```bash
# Check cluster resource availability
kubectl top nodes
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check resource requests in deployment
kubectl describe deployment <deployment-name> -n <namespace> | grep -A 10 "Requests"
```

**Solutions**:

- Scale down other workloads temporarily
- Add more nodes to the cluster
- Reduce resource requests
- Use resource quotas effectively

#### 2. Image Issues

**Problem**: New image cannot be pulled or is failing to start.

```bash
# Check pod events for image issues
kubectl describe pod <pod-name> -n <namespace>

# Check image configuration
kubectl get deployment <deployment-name> -n <namespace> -o yaml | grep image
```

**Solutions**:

- Verify image name and tag
- Check imagePullSecrets configuration
- Test image pull manually
- Ensure image registry is accessible

#### 3. Rolling Update Strategy Issues

**Problem**: Rolling update strategy is preventing successful rollout.

```bash
# Check rolling update strategy
kubectl get deployment <deployment-name> -n <namespace> -o yaml | grep strategy -A 10
```

**Solutions**:

- Adjust `maxUnavailable` and `maxSurge` values
- Consider using `Recreate` strategy for testing
- Ensure sufficient resources for parallel pods

#### 4. Readiness Probe Issues

**Problem**: New pods are not passing readiness checks.

```bash
# Check readiness probe configuration
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Readiness"

# Test readiness endpoint manually
kubectl exec -it <pod-name> -n <namespace> -- curl http://localhost:8080/ready
```

**Solutions**:

- Adjust readiness probe timing
- Fix application readiness endpoint
- Increase probe timeout values

---

## Old Pods Not Terminating

### Symptoms and Diagnosis

- Old replica set still has running pods
- Multiple replica sets exist
- Rollout appears complete but old pods remain

### Troubleshooting Steps

```bash
# Check all replica sets
kubectl get rs -n <namespace>

# Check deployment strategy
kubectl get deployment <deployment-name> -n <namespace> -o yaml | grep strategy -A 10

# Check for stuck pods
kubectl get pods -n <namespace> --show-labels
kubectl describe pod <stuck-pod-name> -n <namespace>
```

### Solutions

#### 1. Manual Pod Cleanup

```bash
# Delete old replica set (if safe)
kubectl delete rs <old-replicaset-name> -n <namespace>

# Force delete stuck pods
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force
```

#### 2. Force Rollout Restart

```bash
# Trigger new rollout
kubectl rollout restart deployment/<deployment-name> -n <namespace>

# Wait for rollout completion
kubectl rollout status deployment/<deployment-name> -n <namespace>
```

#### 3. Check for Finalizers

```bash
# Check if pods have finalizers preventing deletion
kubectl get pod <pod-name> -n <namespace> -o yaml | grep finalizers -A 5
```

---

## Failed Rollout

### Symptoms and Diagnosis

- Rollout stuck or failed
- Error messages in deployment events
- Pods not reaching ready state

### Troubleshooting Steps

```bash
# Check rollout status with timeout
kubectl rollout status deployment/<deployment-name> -n <namespace> --timeout=300s

# Check deployment events
kubectl describe deployment <deployment-name> -n <namespace>

# Check failed pods
kubectl get pods -n <namespace> -l app=<app-label>
kubectl logs <failed-pod> -n <namespace>
```

### Solutions

#### 1. Rollback to Previous Version

```bash
# View rollout history
kubectl rollout history deployment/<deployment-name> -n <namespace>

# Rollback to previous version
kubectl rollout undo deployment/<deployment-name> -n <namespace>

# Rollback to specific revision
kubectl rollout undo deployment/<deployment-name> -n <namespace> --to-revision=<revision-number>
```

#### 2. Pause and Resume Rollout

```bash
# Pause rollout to investigate
kubectl rollout pause deployment/<deployment-name> -n <namespace>

# Make necessary fixes, then resume
kubectl rollout resume deployment/<deployment-name> -n <namespace>
```

---

## Scaling Issues

### Horizontal Scaling Problems

```bash
# Check current replica count
kubectl get deployment <deployment-name> -n <namespace>

# Scale deployment
kubectl scale deployment <deployment-name> --replicas=<number> -n <namespace>

# Check scaling events
kubectl describe deployment <deployment-name> -n <namespace>
```

### Autoscaling Issues

```bash
# Check HPA status
kubectl get hpa -n <namespace>
kubectl describe hpa <hpa-name> -n <namespace>

# Check metrics availability
kubectl top pods -n <namespace>

# Check HPA events
kubectl get events -n <namespace> | grep HorizontalPodAutoscaler
```

### Common Scaling Problems

#### 1. Resource Quotas

```bash
# Check namespace resource quotas
kubectl describe quota -n <namespace>

# Check limit ranges
kubectl describe limitrange -n <namespace>
```

#### 2. Node Capacity

```bash
# Check node capacity and allocation
kubectl describe nodes | grep -A 5 "Capacity\|Allocatable"
```

---

## Resource Management

### CPU and Memory Issues

```bash
# Check resource usage
kubectl top pods -n <namespace>

# Check resource requests and limits
kubectl describe deployment <deployment-name> -n <namespace> | grep -A 10 "Limits\|Requests"

# Check for resource-related events
kubectl get events -n <namespace> | grep -E "FailedScheduling|OOMKilled"
```

### Storage Issues

```bash
# Check PVC status for deployment
kubectl get pvc -n <namespace>

# Check volume mounts in deployment
kubectl get deployment <deployment-name> -n <namespace> -o yaml | grep -A 20 volumes
```

---

## Rollback Procedures

### Safe Rollback Steps

1. **Check current status**:

```bash
kubectl rollout status deployment/<deployment-name> -n <namespace>
kubectl get pods -n <namespace> -l app=<app-label>
```

2. **View rollout history**:

```bash
kubectl rollout history deployment/<deployment-name> -n <namespace>
kubectl rollout history deployment/<deployment-name> -n <namespace> --revision=<number>
```

3. **Perform rollback**:

```bash
kubectl rollout undo deployment/<deployment-name> -n <namespace>
```

4. **Verify rollback**:

```bash
kubectl rollout status deployment/<deployment-name> -n <namespace>
kubectl get pods -n <namespace> -l app=<app-label>
```

### Emergency Rollback

```bash
# Quick rollback to previous version
kubectl rollout undo deployment/<deployment-name> -n <namespace>

# Force rollback by scaling to 0 and back
kubectl scale deployment <deployment-name> --replicas=0 -n <namespace>
kubectl scale deployment <deployment-name> --replicas=<original-count> -n <namespace>
```

---

## Monitoring Deployment Health

### Continuous Monitoring

```bash
# Watch deployment status
kubectl get deployments -n <namespace> -w

# Watch pod status
kubectl get pods -n <namespace> -l app=<app-label> -w

# Monitor events
kubectl get events -n <namespace> -w
```

### Health Checks

```bash
# Check deployment readiness
kubectl get deployment <deployment-name> -n <namespace> -o json | jq '.status.conditions[]'

# Check pod readiness
kubectl get pods -n <namespace> -l app=<app-label> -o json | jq '.items[].status.conditions[]'
```

---

## Quick Reference Commands

```bash
# Deployment management
kubectl create deployment <name> --image=<image> -n <namespace>
kubectl get deployments -n <namespace>
kubectl describe deployment <name> -n <namespace>
kubectl delete deployment <name> -n <namespace>

# Scaling
kubectl scale deployment <name> --replicas=<count> -n <namespace>
kubectl autoscale deployment <name> --min=<min> --max=<max> --cpu-percent=<percent> -n <namespace>

# Rollout management
kubectl rollout status deployment/<name> -n <namespace>
kubectl rollout history deployment/<name> -n <namespace>
kubectl rollout undo deployment/<name> -n <namespace>
kubectl rollout restart deployment/<name> -n <namespace>
kubectl rollout pause deployment/<name> -n <namespace>
kubectl rollout resume deployment/<name> -n <namespace>

# Image updates
kubectl set image deployment/<name> <container>=<image> -n <namespace>

# Resource management
kubectl patch deployment <name> -p '{"spec":{"template":{"spec":{"containers":[{"name":"<container>","resources":{"requests":{"memory":"64Mi","cpu":"250m"},"limits":{"memory":"128Mi","cpu":"500m"}}}]}}}}' -n <namespace>

# Debugging
kubectl get rs -n <namespace>
kubectl get pods -n <namespace> -l app=<app-label>
kubectl logs deployment/<name> -n <namespace>
```

## Best Practices

1. **Use Rolling Updates**: Configure appropriate `maxUnavailable` and `maxSurge` values
2. **Set Resource Limits**: Always define CPU and memory requests/limits
3. **Implement Health Checks**: Use liveness, readiness, and startup probes
4. **Monitor Rollouts**: Always check rollout status after updates
5. **Keep Rollout History**: Maintain revision history for easy rollbacks
6. **Test Before Production**: Use staging environments to test deployments
7. **Use Labels Consistently**: Proper labeling makes troubleshooting easier
8. **Document Changes**: Keep track of deployment changes and reasons

## Common Deployment Patterns

### Blue-Green Deployment

```bash
# Create new deployment with different label
kubectl create deployment <app>-green --image=<new-image> -n <namespace>

# Scale up green deployment
kubectl scale deployment <app>-green --replicas=<count> -n <namespace>

# Update service selector to point to green
kubectl patch service <service-name> -p '{"spec":{"selector":{"version":"green"}}}' -n <namespace>

# Remove blue deployment
kubectl delete deployment <app>-blue -n <namespace>
```

### Canary Deployment

```bash
# Deploy canary version with fewer replicas
kubectl create deployment <app>-canary --image=<new-image> -n <namespace>
kubectl scale deployment <app>-canary --replicas=1 -n <namespace>

# Monitor canary performance
kubectl logs deployment/<app>-canary -n <namespace>

# Gradually shift traffic by scaling
kubectl scale deployment <app>-canary --replicas=<more> -n <namespace>
kubectl scale deployment <app>-stable --replicas=<fewer> -n <namespace>
```

---

## Hands-On Testing Scenarios

Practice these scenarios in your Minikube or kind cluster to gain practical troubleshooting experience.

### Scenario 1: Deployment Stuck Due to Resource Constraints

**Setup the Problem:**

```bash
# Start minikube with limited resources
minikube start --memory=2048 --cpus=2

# Enable metrics server
minikube addons enable metrics-server

# Create a deployment with high resource requests
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-hungry-app
spec:
  replicas: 5
  selector:
    matchLabels:
      app: resource-hungry
  template:
    metadata:
      labels:
        app: resource-hungry
    spec:
      containers:
      - name: app
        image: nginx:alpine
        resources:
          requests:
            memory: "2Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "500m"
EOF
```

**Troubleshoot:**

```bash
# Check deployment status
kubectl get deployments
kubectl describe deployment resource-hungry-app

# Check pod status
kubectl get pods
kubectl describe pod <pending-pod-name>

# Check node resources
kubectl top nodes
kubectl describe nodes
```

**Fix:**

```bash
# Scale down the deployment
kubectl scale deployment resource-hungry-app --replicas=2

# Or reduce resource requests
kubectl patch deployment resource-hungry-app -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","resources":{"requests":{"memory":"256Mi","cpu":"100m"}}}]}}}}'
```

### Scenario 2: Failed Rollout Due to Image Issues

**Setup the Problem:**

```bash
# Create a working deployment
kubectl create deployment image-test-app --image=nginx:alpine

# Update to a non-existent image
kubectl set image deployment/image-test-app nginx=nginx:nonexistent-tag
```

**Troubleshoot:**

```bash
# Check rollout status
kubectl rollout status deployment/image-test-app --timeout=60s

# Check deployment and replica sets
kubectl get deployments
kubectl get rs
kubectl describe rs <new-replicaset-name>

# Check pod events
kubectl get pods
kubectl describe pod <failed-pod-name>
```

**Fix:**

```bash
# Rollback to previous version
kubectl rollout undo deployment/image-test-app

# Or fix the image
kubectl set image deployment/image-test-app nginx=nginx:alpine
```

### Scenario 3: Readiness Probe Causing Rollout Failure

**Setup the Problem:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-test-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: probe-test
  template:
    metadata:
      labels:
        app: probe-test
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /nonexistent-endpoint
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
EOF
```

**Troubleshoot:**

```bash
# Check deployment status
kubectl get deployments
kubectl rollout status deployment/probe-test-app --timeout=60s

# Check pod readiness
kubectl get pods
kubectl describe pod <pod-name>

# Test the readiness probe manually
kubectl exec -it <pod-name> -- curl -f http://localhost/nonexistent-endpoint
```

**Fix:**

```bash
# Fix the readiness probe
kubectl patch deployment probe-test-app -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","readinessProbe":{"httpGet":{"path":"/","port":80}}}]}}}}'
```

### Scenario 4: Rolling Update Strategy Issues

**Setup the Problem:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-update-test
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1  
  selector:
    matchLabels:
      app: rolling-test
  template:
    metadata:
      labels:
        app: rolling-test
    spec:
      containers:
      - name: app
        image: nginx:alpine
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
EOF

# Try to update the image
kubectl set image deployment/rolling-update-test app=nginx:latest
```

**Troubleshoot:**

```bash
# Check why rollout is stuck
kubectl rollout status deployment/rolling-update-test --timeout=60s
kubectl describe deployment rolling-update-test
kubectl get rs
```

**Fix:**

```bash
# Fix the rolling update strategy
kubectl patch deployment rolling-update-test -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":"25%","maxUnavailable":"25%"}}}}'
```

### Scenario 5: Deployment Cleanup Issues

**Setup the Problem:**

```bash
# Create deployment with finalizers
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: finalizer-test
  finalizers:
    - example.com/test-finalizer
spec:
  replicas: 2
  selector:
    matchLabels:
      app: finalizer-test
  template:
    metadata:
      labels:
        app: finalizer-test
    spec:
      containers:
      - name: app
        image: nginx:alpine
EOF

# Try to delete the deployment
kubectl delete deployment finalizer-test
```

**Troubleshoot:**

```bash
# Check why deployment won't delete
kubectl get deployments
kubectl describe deployment finalizer-test
```

**Fix:**

```bash
# Remove finalizers
kubectl patch deployment finalizer-test -p '{"metadata":{"finalizers":[]}}' --type=merge
```

### Complete Testing Script

**Save this as `test-deployments.sh`:**

```bash
#!/bin/bash

echo "=== Kubernetes Deployment Troubleshooting Scenarios ==="
echo "Make sure you have minikube or kind cluster running!"

# Scenario 1: Resource constraints
echo "Scenario 1: Testing resource constraints..."
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-test
spec:
  replicas: 10
  selector:
    matchLabels:
      app: resource-test
  template:
    metadata:
      labels:
        app: resource-test
    spec:
      containers:
      - name: app
        image: nginx:alpine
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
EOF

echo "Check deployment status: kubectl get deployments"
echo "Check pod status: kubectl get pods"
echo "Check node resources: kubectl describe nodes"
echo "Fix: kubectl scale deployment resource-test --replicas=1"
echo ""

# Scenario 2: Image pull issues
echo "Scenario 2: Testing image pull issues..."
kubectl create deployment image-test --image=nginx:nonexistent
echo "Check status: kubectl describe deployment image-test"
echo "Fix: kubectl set image deployment/image-test nginx=nginx:alpine"
echo ""

echo "Run these commands to explore each scenario!"
echo "Cleanup: kubectl delete deployment resource-test image-test"
```

### Practice Exercise

1. **Set up your environment:**
   ```bash
   # Using minikube
   minikube start --memory=4096 --cpus=2
   
   # Or using kind
   kind create cluster --config - <<EOF
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   nodes:
   - role: control-plane
   - role: worker
   EOF
   ```

2. **Run through each scenario** and practice the troubleshooting steps

3. **Create your own scenarios** by modifying the YAML configurations

4. **Clean up:**
   ```bash
   kubectl delete deployments --all
   minikube delete  # or kind delete cluster
   ```

Remember: Deployment troubleshooting often involves understanding the entire application lifecycle, from image building to service exposure.
