# RBAC Troubleshooting Guide

Guide for troubleshooting Kubernetes Role-Based Access Control (RBAC) issues.

## Common RBAC Issues

### Permission Denied Errors

```bash
# Check current user permissions
kubectl auth can-i <verb> <resource> --as=<user> -n <namespace>
kubectl auth can-i --list --as=<user> -n <namespace>

# Check service account permissions
kubectl get serviceaccount -n <namespace>
kubectl describe serviceaccount <sa-name> -n <namespace>

# Check roles and role bindings
kubectl get roles,rolebindings -n <namespace>
kubectl get clusterroles,clusterrolebindings

# Describe specific RBAC resources
kubectl describe role <role-name> -n <namespace>
kubectl describe rolebinding <binding-name> -n <namespace>
```

### Service Account Token Issues

```bash
# Check service account secrets
kubectl get serviceaccount <sa-name> -n <namespace> -o yaml
kubectl get secrets -n <namespace> | grep <sa-name>

# Check pod service account
kubectl get pod <pod-name> -n <namespace> -o yaml | grep serviceAccount
kubectl describe pod <pod-name> -n <namespace> | grep "Service Account"

# Test API access from pod
kubectl exec -it <pod-name> -n <namespace> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

### Role and RoleBinding Issues

```bash
# Check role permissions
kubectl describe role <role-name> -n <namespace>

# Check role binding subjects
kubectl describe rolebinding <binding-name> -n <namespace>

# Verify subject names and namespaces
kubectl get rolebinding <binding-name> -n <namespace> -o yaml
```

## Quick Commands

```bash
# Permission testing
kubectl auth can-i get pods --as=system:serviceaccount:<namespace>:<serviceaccount>
kubectl auth can-i create deployments --as=<user> -n <namespace>
kubectl auth can-i --list -n <namespace>

# RBAC resource management
kubectl get roles,rolebindings -n <namespace>
kubectl get clusterroles,clusterrolebindings
kubectl describe role <role-name> -n <namespace>
kubectl describe rolebinding <binding-name> -n <namespace>

# Service account management
kubectl get serviceaccounts -n <namespace>
kubectl create serviceaccount <sa-name> -n <namespace>
kubectl get secret <sa-secret> -n <namespace> -o yaml

# Creating RBAC resources
kubectl create role <role-name> --verb=get,list --resource=pods -n <namespace>
kubectl create rolebinding <binding-name> --role=<role-name> --serviceaccount=<namespace>:<sa-name> -n <namespace>
```

---

## Hands-On Testing Scenarios

Practice these scenarios in your Minikube or kind cluster to gain practical RBAC troubleshooting experience.

### Scenario 1: Service Account Without Permissions

**Setup the Problem:**

```bash
# Create namespace and service account
kubectl create namespace rbac-test
kubectl create serviceaccount limited-sa -n rbac-test

# Create pod using the service account
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: rbac-test-pod
  namespace: rbac-test
spec:
  serviceAccountName: limited-sa
  containers:
  - name: kubectl-container
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
EOF

kubectl wait --for=condition=Ready pod/rbac-test-pod -n rbac-test --timeout=60s
```

**Troubleshoot:**

```bash
# Test pod's permissions (should fail)
kubectl exec rbac-test-pod -n rbac-test -- kubectl get pods
kubectl exec rbac-test-pod -n rbac-test -- kubectl get secrets

# Check what the service account can do
kubectl auth can-i get pods --as=system:serviceaccount:rbac-test:limited-sa -n rbac-test
kubectl auth can-i get secrets --as=system:serviceaccount:rbac-test:limited-sa -n rbac-test

# List current permissions
kubectl auth can-i --list --as=system:serviceaccount:rbac-test:limited-sa -n rbac-test

# Check existing roles and bindings
kubectl get roles,rolebindings -n rbac-test
```

**Fix:**

```bash
# Create role with specific permissions
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rbac-test
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
EOF

# Create role binding
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: rbac-test
subjects:
- kind: ServiceAccount
  name: limited-sa
  namespace: rbac-test
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF

# Test permissions again
kubectl exec rbac-test-pod -n rbac-test -- kubectl get pods
kubectl auth can-i get pods --as=system:serviceaccount:rbac-test:limited-sa -n rbac-test
```

### Scenario 2: ClusterRole vs Role Confusion

**Setup the Problem:**

```bash
# Create service account that needs cluster-wide access
kubectl create serviceaccount cluster-viewer -n rbac-test

# Create a Role (namespace-scoped) instead of ClusterRole
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rbac-test
  name: node-viewer  # This should be a ClusterRole
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: node-viewer-binding
  namespace: rbac-test
subjects:
- kind: ServiceAccount
  name: cluster-viewer
  namespace: rbac-test
roleRef:
  kind: Role
  name: node-viewer
  apiGroup: rbac.authorization.k8s.io
EOF

# Create pod to test
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cluster-test-pod
  namespace: rbac-test
spec:
  serviceAccountName: cluster-viewer
  containers:
  - name: kubectl-container
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
EOF

kubectl wait --for=condition=Ready pod/cluster-test-pod -n rbac-test --timeout=60s
```

**Troubleshoot:**

```bash
# Test node access (should fail)
kubectl exec cluster-test-pod -n rbac-test -- kubectl get nodes

# Check permissions
kubectl auth can-i get nodes --as=system:serviceaccount:rbac-test:cluster-viewer

# Understand the issue - Roles are namespace-scoped, nodes are cluster-scoped
kubectl explain role.rules.resources
kubectl explain clusterrole.rules.resources
```

**Fix:**

```bash
# Delete the incorrect Role and RoleBinding
kubectl delete role node-viewer -n rbac-test
kubectl delete rolebinding node-viewer-binding -n rbac-test

# Create correct ClusterRole and ClusterRoleBinding
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-viewer
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-viewer-binding
subjects:
- kind: ServiceAccount
  name: cluster-viewer
  namespace: rbac-test
roleRef:
  kind: ClusterRole
  name: node-viewer
  apiGroup: rbac.authorization.k8s.io
EOF

# Test again
kubectl exec cluster-test-pod -n rbac-test -- kubectl get nodes
```

### Scenario 3: Missing API Groups in Permissions

**Setup the Problem:**

```bash
# Create service account for deployment management
kubectl create serviceaccount deployment-manager -n rbac-test

# Create role with incorrect API groups
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rbac-test
  name: deployment-manager
rules:
- apiGroups: [""]  # Wrong API group for deployments
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployment-manager-binding
  namespace: rbac-test
subjects:
- kind: ServiceAccount
  name: deployment-manager
  namespace: rbac-test
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
EOF

# Create pod to test
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: deployment-manager-pod
  namespace: rbac-test
spec:
  serviceAccountName: deployment-manager
  containers:
  - name: kubectl-container
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
EOF

kubectl wait --for=condition=Ready pod/deployment-manager-pod -n rbac-test --timeout=60s
```

**Troubleshoot:**

```bash
# Test deployment operations (should fail)
kubectl exec deployment-manager-pod -n rbac-test -- kubectl get deployments
kubectl exec deployment-manager-pod -n rbac-test -- kubectl create deployment test --image=nginx

# Check permissions
kubectl auth can-i get deployments --as=system:serviceaccount:rbac-test:deployment-manager -n rbac-test

# Find correct API group for deployments
kubectl api-resources | grep deployments
kubectl explain deployment
```

**Fix:**

```bash
# Update role with correct API group
kubectl patch role deployment-manager -n rbac-test -p '{"rules":[{"apiGroups":["apps"],"resources":["deployments"],"verbs":["get","list","create","update","delete"]}]}'

# Test again
kubectl exec deployment-manager-pod -n rbac-test -- kubectl get deployments
kubectl exec deployment-manager-pod -n rbac-test -- kubectl create deployment test-deploy --image=nginx:alpine
```

### Scenario 4: Overly Restrictive Resource Names

**Setup the Problem:**

```bash
# Create role with specific resource names
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rbac-test
  name: specific-pod-access
rules:
- apiGroups: [""]
  resources: ["pods"]
  resourceNames: ["allowed-pod"]  # Only specific pod allowed
  verbs: ["get", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: specific-pod-binding
  namespace: rbac-test
subjects:
- kind: ServiceAccount
  name: limited-sa
  namespace: rbac-test
roleRef:
  kind: Role
  name: specific-pod-access
  apiGroup: rbac.authorization.k8s.io
EOF

# Create test pods
kubectl run allowed-pod --image=nginx:alpine -n rbac-test
kubectl run denied-pod --image=nginx:alpine -n rbac-test

kubectl wait --for=condition=Ready pod/allowed-pod -n rbac-test --timeout=60s
kubectl wait --for=condition=Ready pod/denied-pod -n rbac-test --timeout=60s
```

**Troubleshoot:**

```bash
# Test access to specific pod (should work)
kubectl auth can-i get pod allowed-pod --as=system:serviceaccount:rbac-test:limited-sa -n rbac-test

# Test access to other pod (should fail)
kubectl auth can-i get pod denied-pod --as=system:serviceaccount:rbac-test:limited-sa -n rbac-test

# Test listing pods (should fail - resourceNames doesn't work with list)
kubectl auth can-i list pods --as=system:serviceaccount:rbac-test:limited-sa -n rbac-test

# Test from pod
kubectl exec rbac-test-pod -n rbac-test -- kubectl get pod allowed-pod
kubectl exec rbac-test-pod -n rbac-test -- kubectl get pod denied-pod || echo "Access denied"
kubectl exec rbac-test-pod -n rbac-test -- kubectl get pods || echo "List denied"
```

**Fix:**

```bash
# Update role to allow listing (remove resourceNames for list verb)
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rbac-test
  name: specific-pod-access
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list"]  # List without resourceNames restriction
- apiGroups: [""]
  resources: ["pods"]
  resourceNames: ["allowed-pod"]
  verbs: ["get", "delete"]  # Specific operations on named resource
EOF

# Test listing now works
kubectl exec rbac-test-pod -n rbac-test -- kubectl get pods
```

### Scenario 5: ServiceAccount Token Issues

**Setup the Problem:**

```bash
# Create service account
kubectl create serviceaccount token-test-sa -n rbac-test

# Create pod without automounting service account token
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-token-pod
  namespace: rbac-test
spec:
  serviceAccountName: token-test-sa
  automountServiceAccountToken: false  # Disable token mounting
  containers:
  - name: test-container
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
EOF

kubectl wait --for=condition=Ready pod/no-token-pod -n rbac-test --timeout=60s
```

**Troubleshoot:**

```bash
# Check if token is mounted
kubectl exec no-token-pod -n rbac-test -- ls /var/run/secrets/kubernetes.io/serviceaccount/ || echo "No token mounted"

# Try kubectl operations (should fail)
kubectl exec no-token-pod -n rbac-test -- kubectl get pods || echo "Authentication failed"

# Check service account secrets
kubectl get serviceaccount token-test-sa -n rbac-test -o yaml
kubectl get secrets -n rbac-test | grep token-test-sa

# Check for token secret
kubectl describe serviceaccount token-test-sa -n rbac-test
```

**Fix:**

```bash
# Enable token mounting
kubectl patch pod no-token-pod -n rbac-test -p '{"spec":{"automountServiceAccountToken":true}}'

# Or recreate pod with token mounting enabled
kubectl delete pod no-token-pod -n rbac-test

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: with-token-pod
  namespace: rbac-test
spec:
  serviceAccountName: token-test-sa
  automountServiceAccountToken: true
  containers:
  - name: test-container
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
EOF

kubectl wait --for=condition=Ready pod/with-token-pod -n rbac-test --timeout=60s

# Verify token is mounted
kubectl exec with-token-pod -n rbac-test -- ls /var/run/secrets/kubernetes.io/serviceaccount/
kubectl exec with-token-pod -n rbac-test -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

### Scenario 6: Cross-Namespace RBAC Issues

**Setup the Problem:**

```bash
# Create another namespace
kubectl create namespace rbac-test-2

# Create service account in first namespace
kubectl create serviceaccount cross-ns-sa -n rbac-test

# Create role in second namespace
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rbac-test-2
  name: cross-ns-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cross-ns-binding
  namespace: rbac-test-2
subjects:
- kind: ServiceAccount
  name: cross-ns-sa
  namespace: rbac-test  # Different namespace
roleRef:
  kind: Role
  name: cross-ns-role
  apiGroup: rbac.authorization.k8s.io
EOF

# Create test pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cross-ns-pod
  namespace: rbac-test
spec:
  serviceAccountName: cross-ns-sa
  containers:
  - name: kubectl-container
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
EOF

kubectl wait --for=condition=Ready pod/cross-ns-pod -n rbac-test --timeout=60s

# Create some pods in the target namespace
kubectl run target-pod-1 --image=nginx:alpine -n rbac-test-2
kubectl run target-pod-2 --image=nginx:alpine -n rbac-test-2
```

**Troubleshoot:**

```bash
# Test cross-namespace access
kubectl exec cross-ns-pod -n rbac-test -- kubectl get pods -n rbac-test-2

# Check permissions
kubectl auth can-i get pods --as=system:serviceaccount:rbac-test:cross-ns-sa -n rbac-test-2
kubectl auth can-i get pods --as=system:serviceaccount:rbac-test:cross-ns-sa -n rbac-test

# Verify role binding
kubectl describe rolebinding cross-ns-binding -n rbac-test-2
```

**Fix:**

```bash
# This should already work - cross-namespace RBAC is allowed
# Test it
kubectl exec cross-ns-pod -n rbac-test -- kubectl get pods -n rbac-test-2

# If issues, check the binding syntax
kubectl get rolebinding cross-ns-binding -n rbac-test-2 -o yaml
```

### Complete Testing Script

**Save this as `test-rbac.sh`:**

```bash
#!/bin/bash

echo "=== Kubernetes RBAC Troubleshooting Scenarios ==="

# Create test namespace
kubectl create namespace rbac-demo 2>/dev/null || echo "rbac-demo namespace exists"

echo "=== Scenario 1: Basic ServiceAccount Permissions ==="
# Create service account
kubectl create serviceaccount demo-sa -n rbac-demo 2>/dev/null || echo "ServiceAccount exists"

# Test current permissions
echo "Current permissions for demo-sa:"
kubectl auth can-i --list --as=system:serviceaccount:rbac-demo:demo-sa -n rbac-demo

echo ""
echo "=== Scenario 2: Creating Pod Reader Role ==="
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rbac-demo
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: rbac-demo
subjects:
- kind: ServiceAccount
  name: demo-sa
  namespace: rbac-demo
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF

echo "Updated permissions for demo-sa:"
kubectl auth can-i get pods --as=system:serviceaccount:rbac-demo:demo-sa -n rbac-demo
kubectl auth can-i create pods --as=system:serviceaccount:rbac-demo:demo-sa -n rbac-demo

echo ""
echo "=== Scenario 3: Testing with Real Pod ==="
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: rbac-test-pod
  namespace: rbac-demo
spec:
  serviceAccountName: demo-sa
  containers:
  - name: kubectl-container
    image: bitnami/kubectl:latest
    command: ["sleep", "300"]
EOF

echo "Waiting for pod to be ready..."
kubectl wait --for=condition=Ready pod/rbac-test-pod -n rbac-demo --timeout=60s

echo "Testing permissions from within pod..."
kubectl exec rbac-test-pod -n rbac-demo -- kubectl get pods
kubectl exec rbac-test-pod -n rbac-demo -- kubectl get secrets 2>/dev/null || echo "No access to secrets (expected)"

echo ""
echo "=== Current RBAC Resources ==="
echo "Roles:"
kubectl get roles -n rbac-demo

echo ""
echo "RoleBindings:"
kubectl get rolebindings -n rbac-demo

echo ""
echo "ServiceAccounts:"
kubectl get serviceaccounts -n rbac-demo

echo ""
echo "=== Cleanup Commands ==="
echo "kubectl delete namespace rbac-demo"
```

### Practice Exercise

1. **Set up RBAC environment:**
   ```bash
   # Create multiple namespaces and service accounts
   # Practice different role combinations
   # Test ClusterRole vs Role scenarios
   ```

2. **Test permission boundaries:**
   ```bash
   # Create minimal permissions and gradually add
   # Test API group requirements
   # Understand resource vs subresource permissions
   ```

3. **Debug permission issues:**
   ```bash
   # Use `kubectl auth can-i` extensively
   # Check role bindings and subjects
   # Understand token mounting behavior
   ```

4. **Practice real-world scenarios:**
   ```bash
   # Create developer roles
   # Set up CI/CD service accounts
   # Configure namespace isolation
   ```

5. **Clean up:**
   ```bash
   kubectl delete namespace rbac-test rbac-test-2 rbac-demo 2>/dev/null
   kubectl delete clusterrole node-viewer 2>/dev/null
   kubectl delete clusterrolebinding node-viewer-binding 2>/dev/null
   ```

### Advanced Scenarios

**Aggregated ClusterRoles:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: aggregated-role
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: []  # Rules filled in by aggregation
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-monitoring
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
EOF
```

**User Impersonation:**

```bash
# Test user impersonation (requires appropriate permissions)
kubectl auth can-i create pods --as=jane
kubectl get pods --as=jane --as-group=developers

# Create role for impersonation
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: impersonator
rules:
- apiGroups: [""]
  resources: ["users", "groups", "serviceaccounts"]
  verbs: ["impersonate"]
EOF
```

**Webhook Authentication:**

```bash
# Check authentication webhook configuration
kubectl get validatingadmissionwebhooks
kubectl get mutatingadmissionwebhooks

# Test OIDC token authentication (if configured)
kubectl auth can-i get pods --token="<oidc-token>"
```

Remember: RBAC is additive - permissions are granted, not denied. Always follow the principle of least privilege and regularly audit your RBAC configurations.
