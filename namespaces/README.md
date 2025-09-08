# Namespace Troubleshooting Guide

Guide for troubleshooting Kubernetes Namespace issues and resource isolation.

## Common Namespace Issues

### Namespace Stuck in Terminating

```bash
# Check namespace status
kubectl get namespace <namespace-name> -o yaml

# Check for remaining resources
kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found -n <namespace-name>

# Check for finalizers
kubectl get namespace <namespace-name> -o yaml | grep finalizers -A 5

# Force delete namespace (use with caution)
kubectl get namespace <namespace-name> -o json | jq '.spec.finalizers = []' | kubectl replace --raw "/api/v1/namespaces/<namespace-name>/finalize" -f -
```

### Resource Quotas and Limits

```bash
# Check namespace resource quotas
kubectl get resourcequota -n <namespace-name>
kubectl describe resourcequota -n <namespace-name>

# Check limit ranges
kubectl get limitrange -n <namespace-name>
kubectl describe limitrange -n <namespace-name>

# Check current resource usage
kubectl describe namespace <namespace-name>
kubectl top pods -n <namespace-name>
```

### Cross-Namespace Communication

```bash
# Test cross-namespace service communication
kubectl exec -it <pod-name> -n <source-namespace> -- nslookup <service-name>.<target-namespace>.svc.cluster.local

# Check network policies affecting namespaces
kubectl get networkpolicies --all-namespaces
kubectl describe networkpolicy <policy-name> -n <namespace-name>
```

## Quick Commands

```bash
# Namespace management
kubectl get namespaces
kubectl describe namespace <namespace-name>
kubectl create namespace <namespace-name>
kubectl delete namespace <namespace-name>

# Resource management
kubectl get all -n <namespace-name>
kubectl get events -n <namespace-name>
kubectl get resourcequota -n <namespace-name>
kubectl get limitrange -n <namespace-name>

# Cleanup
kubectl delete all --all -n <namespace-name>
kubectl get all -n <namespace-name>  # Verify cleanup
```

---

## Hands-On Testing Scenarios

Practice these scenarios in your Minikube or kind cluster to gain practical Namespace troubleshooting experience.

### Scenario 1: Namespace Termination Stuck

**Setup the Problem:**

```bash
# Create namespace with resources
kubectl create namespace stuck-namespace

# Create some resources in the namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-config
  namespace: stuck-namespace
data:
  key: value
---
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
  namespace: stuck-namespace
type: Opaque
data:
  password: cGFzc3dvcmQ=
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
  namespace: stuck-namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: app
        image: nginx:alpine
EOF

# Add finalizer to simulate stuck deletion
kubectl patch configmap test-config -n stuck-namespace -p '{"metadata":{"finalizers":["example.com/custom-finalizer"]}}'

# Try to delete namespace (it will get stuck)
kubectl delete namespace stuck-namespace
```

**Troubleshoot:**

```bash
# Check namespace status
kubectl get namespaces
kubectl describe namespace stuck-namespace

# Check what's preventing deletion
kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found -n stuck-namespace

# Check for finalizers
kubectl get configmap test-config -n stuck-namespace -o yaml | grep finalizers -A 5
```

**Fix:**

```bash
# Remove problematic finalizers
kubectl patch configmap test-config -n stuck-namespace -p '{"metadata":{"finalizers":null}}' --type=merge

# Verify namespace deletion completes
kubectl get namespaces | grep stuck-namespace
```

### Scenario 2: Cross-Namespace Service Discovery Issues

**Setup the Problem:**

```bash
# Create two namespaces
kubectl create namespace frontend
kubectl create namespace backend

# Create backend service
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
  namespace: backend
  labels:
    app: backend
spec:
  containers:
  - name: backend
    image: nginx:alpine
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: backend
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
EOF

# Create frontend pod that tries to access backend
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  namespace: frontend
spec:
  containers:
  - name: frontend
    image: busybox
    command: ["sleep", "3600"]
EOF

kubectl wait --for=condition=Ready pod/backend-pod -n backend --timeout=60s
kubectl wait --for=condition=Ready pod/frontend-pod -n frontend --timeout=60s
```

**Troubleshoot:**

```bash
# Test incorrect DNS resolution (this will fail)
kubectl exec frontend-pod -n frontend -- nslookup backend-service

# Test correct cross-namespace DNS
kubectl exec frontend-pod -n frontend -- nslookup backend-service.backend.svc.cluster.local

# Check service endpoints
kubectl get endpoints backend-service -n backend

# Test connectivity
kubectl exec frontend-pod -n frontend -- wget -qO- http://backend-service.backend.svc.cluster.local
```

**Fix:**

```bash
# The fix is using the correct FQDN
echo "Use: backend-service.backend.svc.cluster.local"
echo "Not: backend-service"

# Verify working connection
kubectl exec frontend-pod -n frontend -- wget -qO- http://backend-service.backend.svc.cluster.local
```

### Scenario 3: Resource Quota Violations

**Setup the Problem:**

```bash
# Create namespace with resource quota
kubectl create namespace quota-test

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-test-quota
  namespace: quota-test
spec:
  hard:
    requests.cpu: "100m"
    requests.memory: "128Mi"
    limits.cpu: "200m"
    limits.memory: "256Mi"
    pods: "2"
EOF

# Try to create pods that exceed quota
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quota-exceeding-deployment
  namespace: quota-test
spec:
  replicas: 3  # This exceeds the pod quota of 2
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
            cpu: "150m"  # This exceeds the CPU quota
            memory: "100Mi"
          limits:
            cpu: "300m"  # This exceeds the CPU limit quota
            memory: "200Mi"
EOF
```

**Troubleshoot:**

```bash
# Check resource quota status
kubectl describe resourcequota quota-test-quota -n quota-test

# Check pod creation issues
kubectl get pods -n quota-test
kubectl describe deployment quota-exceeding-deployment -n quota-test

# Check events for quota violations
kubectl get events -n quota-test --sort-by=.metadata.creationTimestamp
```

**Fix:**

```bash
# Update deployment to comply with quota
kubectl patch deployment quota-exceeding-deployment -n quota-test -p '{"spec":{"replicas":2,"template":{"spec":{"containers":[{"name":"app","resources":{"requests":{"cpu":"50m","memory":"64Mi"},"limits":{"cpu":"100m","memory":"128Mi"}}}]}}}}'

# Verify fix
kubectl get pods -n quota-test
kubectl describe resourcequota quota-test-quota -n quota-test
```

### Scenario 4: LimitRange Constraint Violations

**Setup the Problem:**

```bash
# Create namespace with limit ranges
kubectl create namespace limitrange-test

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: limitrange-test-limits
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
EOF

# Try to create pod that violates limits
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: violating-pod
  namespace: limitrange-test
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        cpu: "5m"      # Below minimum
        memory: "512Mi" # Above maximum
      limits:
        cpu: "300m"    # Above maximum
        memory: "1Gi"   # Above maximum
EOF
```

**Troubleshoot:**

```bash
# Check limit range
kubectl describe limitrange limitrange-test-limits -n limitrange-test

# Check pod creation failure
kubectl get pods -n limitrange-test
kubectl describe pod violating-pod -n limitrange-test

# Check events
kubectl get events -n limitrange-test --sort-by=.metadata.creationTimestamp
```

**Fix:**

```bash
# Create pod with compliant resources
kubectl delete pod violating-pod -n limitrange-test

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

### Scenario 5: RBAC and Namespace Isolation

**Setup the Problem:**

```bash
# Create namespaces
kubectl create namespace team-a
kubectl create namespace team-b

# Create service account for team-a
kubectl create serviceaccount team-a-user -n team-a

# Create role that only works in team-a namespace
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-a
  name: team-a-role
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "create", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-binding
  namespace: team-a
subjects:
- kind: ServiceAccount
  name: team-a-user
  namespace: team-a
roleRef:
  kind: Role
  name: team-a-role
  apiGroup: rbac.authorization.k8s.io
EOF

# Create pods in both namespaces
kubectl run test-pod-a --image=nginx:alpine -n team-a
kubectl run test-pod-b --image=nginx:alpine -n team-b
```

**Troubleshoot:**

```bash
# Test access from team-a service account
kubectl auth can-i get pods --as=system:serviceaccount:team-a:team-a-user -n team-a
kubectl auth can-i get pods --as=system:serviceaccount:team-a:team-a-user -n team-b

# Check role and rolebinding
kubectl describe role team-a-role -n team-a
kubectl describe rolebinding team-a-binding -n team-a

# Check what team-a-user can access
kubectl auth can-i --list --as=system:serviceaccount:team-a:team-a-user -n team-a
kubectl auth can-i --list --as=system:serviceaccount:team-a:team-a-user -n team-b
```

**Fix:**

```bash
# This is working as intended - RBAC is properly isolating namespaces
echo "RBAC is working correctly:"
echo "✓ team-a-user can access team-a namespace"
echo "✗ team-a-user cannot access team-b namespace"

# If you need cross-namespace access, create a ClusterRole instead
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cross-namespace-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: team-a-cross-namespace
subjects:
- kind: ServiceAccount
  name: team-a-user
  namespace: team-a
roleRef:
  kind: ClusterRole
  name: cross-namespace-reader
  apiGroup: rbac.authorization.k8s.io
EOF

# Verify cross-namespace access
kubectl auth can-i get pods --as=system:serviceaccount:team-a:team-a-user -n team-b
```

### Scenario 6: NetworkPolicy Namespace Isolation

**Setup the Problem:**

```bash
# Create namespaces
kubectl create namespace isolated-ns
kubectl create namespace public-ns

# Create pods in both namespaces
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: isolated-pod
  namespace: isolated-ns
  labels:
    app: isolated
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
  name: isolated-service
  namespace: isolated-ns
spec:
  selector:
    app: isolated
  ports:
  - port: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: public-pod
  namespace: public-ns
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
EOF

# Create restrictive network policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: isolated-ns
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: isolated-ns
EOF

kubectl wait --for=condition=Ready pod/isolated-pod -n isolated-ns --timeout=60s
kubectl wait --for=condition=Ready pod/public-pod -n public-ns --timeout=60s
```

**Troubleshoot:**

```bash
# Test connectivity (should fail with network policy)
kubectl exec public-pod -n public-ns -- wget -qO- --timeout=5 http://isolated-service.isolated-ns.svc.cluster.local || echo "Connection blocked by NetworkPolicy"

# Check network policies
kubectl get networkpolicies -n isolated-ns
kubectl describe networkpolicy deny-cross-namespace -n isolated-ns

# Test internal namespace connectivity (should work)
kubectl run test-internal -n isolated-ns --image=busybox --rm -it --restart=Never -- wget -qO- http://isolated-service.isolated-ns.svc.cluster.local
```

**Fix:**

```bash
# Allow specific namespace access
kubectl label namespace public-ns name=allowed-ns

kubectl patch networkpolicy deny-cross-namespace -n isolated-ns -p '{"spec":{"ingress":[{"from":[{"namespaceSelector":{"matchLabels":{"name":"isolated-ns"}}},{"namespaceSelector":{"matchLabels":{"name":"allowed-ns"}}}]}]}}'

# Test connectivity (should now work)
kubectl exec public-pod -n public-ns -- wget -qO- http://isolated-service.isolated-ns.svc.cluster.local
```

### Complete Testing Script

**Save this as `test-namespaces.sh`:**

```bash
#!/bin/bash

echo "=== Kubernetes Namespace Troubleshooting Scenarios ==="

echo "=== Scenario 1: Cross-Namespace Communication Test ==="
echo "Creating frontend and backend namespaces..."

kubectl create namespace frontend 2>/dev/null || echo "frontend namespace exists"
kubectl create namespace backend 2>/dev/null || echo "backend namespace exists"

# Create backend service
kubectl run backend-app --image=nginx:alpine --port=80 -n backend
kubectl expose pod backend-app --port=80 --name=backend-svc -n backend

# Create frontend test pod
kubectl run frontend-test --image=busybox --command -n frontend -- sleep 3600

echo "Test cross-namespace DNS:"
echo "kubectl exec frontend-test -n frontend -- nslookup backend-svc.backend.svc.cluster.local"
echo ""

echo "=== Scenario 2: Resource Quota Test ==="
kubectl create namespace quota-demo 2>/dev/null || echo "quota-demo namespace exists"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: demo-quota
  namespace: quota-demo
spec:
  hard:
    pods: "2"
    requests.cpu: "100m"
    requests.memory: "128Mi"
EOF

echo "Created quota allowing only 2 pods with 100m CPU and 128Mi memory"
echo "Try creating 3 pods with: kubectl run test-{1,2,3} --image=nginx --requests=cpu=50m,memory=64Mi -n quota-demo"
echo ""

echo "=== Cleanup Commands ==="
echo "kubectl delete namespace frontend backend quota-demo isolated-ns public-ns team-a team-b limitrange-test quota-test stuck-namespace"
```

### Practice Exercise

1. **Set up namespace isolation:**
   ```bash
   # Create development and production namespaces
   kubectl create namespace development
   kubectl create namespace production
   
   # Set up appropriate RBAC and resource quotas
   ```

2. **Test cross-namespace service discovery:**
   ```bash
   # Create services in different namespaces
   # Test DNS resolution patterns
   # Understand service FQDN structure
   ```

3. **Practice resource management:**
   ```bash
   # Set up ResourceQuotas and LimitRanges
   # Test quota violations
   # Monitor resource usage across namespaces
   ```

4. **Network policy testing:**
   ```bash
   # Create network policies for namespace isolation
   # Test connectivity between namespaces
   # Understand ingress and egress rules
   ```

5. **Clean up:**
   ```bash
   kubectl delete namespace frontend backend quota-demo isolated-ns public-ns team-a team-b limitrange-test quota-test stuck-namespace 2>/dev/null
   ```

### Advanced Scenarios

**Namespace-scoped Custom Resources:**

```bash
# Create namespace-scoped custom resource
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applications.example.com
spec:
  group: example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              name:
                type: string
  scope: Namespaced
  names:
    plural: applications
    singular: application
    kind: Application
EOF
```

**Multi-tenant Resource Isolation:**

```bash
# Create tenant-specific namespaces with proper labeling
for tenant in tenant1 tenant2; do
  kubectl create namespace $tenant
  kubectl label namespace $tenant tenant=$tenant
  
  # Create tenant-specific network policies
  cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-isolation
  namespace: $tenant
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tenant: $tenant
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          tenant: $tenant
EOF
done
```

Remember: Namespaces provide logical isolation but not complete security isolation. Use them with RBAC, network policies, and resource quotas for comprehensive multi-tenancy.
