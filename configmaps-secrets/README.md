# ConfigMap and Secret Troubleshooting Guide

Comprehensive guide for troubleshooting Kubernetes ConfigMap and Secret issues.

## Table of Contents

1. [ConfigMap Issues](#configmap-issues)
2. [Secret Issues](#secret-issues)
3. [Volume Mount Problems](#volume-mount-problems)
4. [Environment Variable Issues](#environment-variable-issues)
5. [Encoding and Data Problems](#encoding-and-data-problems)
6. [Permission and RBAC Issues](#permission-and-rbac-issues)
7. [Update and Reload Issues](#update-and-reload-issues)
8. [Quick Reference Commands](#quick-reference-commands)

---

## ConfigMap Issues

### ConfigMap Not Found

```bash
# Check if ConfigMap exists
kubectl get configmaps -n <namespace>
kubectl describe configmap <configmap-name> -n <namespace>

# Check ConfigMap data
kubectl get configmap <configmap-name> -n <namespace> -o yaml
```

### ConfigMap Data Not Loading

```bash
# Check pod volume configuration
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 20 volumes

# Check volume mounts
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 20 volumeMounts

# Verify data is mounted correctly
kubectl exec -it <pod-name> -n <namespace> -- ls -la /path/to/configmap
kubectl exec -it <pod-name> -n <namespace> -- cat /path/to/configmap/<key>
```

### ConfigMap Update Not Reflected

```bash
# Check if ConfigMap is updated
kubectl get configmap <configmap-name> -n <namespace> -o yaml

# Check mounted file content
kubectl exec -it <pod-name> -n <namespace> -- cat /path/to/configmap/<key>

# Restart pod to pick up changes (for volume mounts)
kubectl delete pod <pod-name> -n <namespace>

# For environment variables, restart deployment
kubectl rollout restart deployment <deployment-name> -n <namespace>
```

---

## Secret Issues

### Secret Not Found or Inaccessible

```bash
# Check if Secret exists
kubectl get secrets -n <namespace>
kubectl describe secret <secret-name> -n <namespace>

# Check Secret type and data
kubectl get secret <secret-name> -n <namespace> -o yaml
```

### Secret Data Encoding Issues

```bash
# Check Secret data encoding
kubectl get secret <secret-name> -n <namespace> -o yaml | grep -A 10 data

# Decode base64 Secret values
kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.<key>}' | base64 -d

# Create Secret with proper encoding
kubectl create secret generic <secret-name> \
  --from-literal=username=<username> \
  --from-literal=password=<password> \
  -n <namespace>
```

### Docker Registry Secret Issues

```bash
# Check imagePullSecrets configuration
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 5 imagePullSecrets

# Check if Docker registry secret exists
kubectl get secret <registry-secret> -n <namespace>

# Create Docker registry secret
kubectl create secret docker-registry <secret-name> \
  --docker-server=<registry-server> \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n <namespace>

# Test image pull with secret
kubectl patch serviceaccount default -p '{"imagePullSecrets":[{"name":"<secret-name>"}]}' -n <namespace>
```

---

## Volume Mount Problems

### Files Not Appearing in Container

```bash
# Check volume and volumeMount configuration
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Volumes:\|Mounts:"

# Check if mount path exists
kubectl exec -it <pod-name> -n <namespace> -- ls -la /path/to/mount/

# Check mount point permissions
kubectl exec -it <pod-name> -n <namespace> -- ls -ld /path/to/mount/

# Check if files are present
kubectl exec -it <pod-name> -n <namespace> -- find /path/to/mount/ -type f
```

### Wrong File Permissions

```bash
# Check file permissions in mount
kubectl exec -it <pod-name> -n <namespace> -- ls -la /path/to/mount/

# Check pod security context
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 10 securityContext

# Check if defaultMode is set
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 5 defaultMode
```

### Subpath Issues

```bash
# Check subPath configuration
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 5 subPath

# Verify specific key mounting
kubectl exec -it <pod-name> -n <namespace> -- cat /path/to/file

# Test without subPath to verify ConfigMap data
kubectl exec -it <pod-name> -n <namespace> -- ls -la /path/to/mount/
```

---

## Environment Variable Issues

### Environment Variables Not Set

```bash
# Check environment variable configuration
kubectl describe pod <pod-name> -n <namespace> | grep -A 20 "Environment:"

# Check actual environment variables in container
kubectl exec -it <pod-name> -n <namespace> -- env | grep <VAR_NAME>

# Check configMapRef and secretRef
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 10 envFrom
```

### Environment Variable Value Issues

```bash
# Check ConfigMap/Secret key exists
kubectl get configmap <configmap-name> -n <namespace> -o yaml | grep -A 10 data
kubectl get secret <secret-name> -n <namespace> -o yaml | grep -A 10 data

# Verify key name matches
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "valueFrom"

# Test environment variable in container
kubectl exec -it <pod-name> -n <namespace> -- echo $<VAR_NAME>
```

---

## Encoding and Data Problems

### Base64 Encoding Issues

```bash
# Check if Secret data is properly encoded
kubectl get secret <secret-name> -n <namespace> -o yaml

# Decode and verify Secret content
echo "<base64-encoded-value>" | base64 -d

# Create Secret with correct encoding
echo -n "<plain-text-value>" | base64

# Create Secret from file
kubectl create secret generic <secret-name> --from-file=<key>=<file-path> -n <namespace>
```

### Binary Data Issues

```bash
# For binary data, use binaryData field
kubectl get secret <secret-name> -n <namespace> -o yaml | grep -A 5 binaryData

# Create Secret with binary data
kubectl create secret generic <secret-name> --from-file=<binary-file> -n <namespace>
```

### ConfigMap Size Limits

```bash
# Check ConfigMap size (limit: 1MB)
kubectl get configmap <configmap-name> -n <namespace> -o yaml | wc -c

# Split large configurations
# Create multiple ConfigMaps if needed
```

---

## Permission and RBAC Issues

### Service Account Access

```bash
# Check pod service account
kubectl get pod <pod-name> -n <namespace> -o yaml | grep serviceAccount

# Check service account permissions
kubectl auth can-i get configmaps --as=system:serviceaccount:<namespace>:<serviceaccount> -n <namespace>
kubectl auth can-i get secrets --as=system:serviceaccount:<namespace>:<serviceaccount> -n <namespace>

# Check RBAC bindings
kubectl get rolebindings,clusterrolebindings -n <namespace>
kubectl describe rolebinding <binding-name> -n <namespace>
```

### Cross-Namespace Access

```bash
# ConfigMaps and Secrets are namespace-scoped
# Check if trying to access from different namespace
kubectl get configmap <configmap-name> -n <target-namespace>

# Copy ConfigMap to correct namespace if needed
kubectl get configmap <configmap-name> -n <source-namespace> -o yaml | \
  sed 's/namespace: <source-namespace>/namespace: <target-namespace>/' | \
  kubectl apply -f -
```

---

## Update and Reload Issues

### Automatic Updates Not Working

```bash
# Check if using projected volumes (updates automatically)
kubectl get pod <pod-name> -n <namespace> -o yaml | grep projected

# For regular volume mounts, updates are automatic but may take time
# Check kubelet sync period (default: 1 minute)

# For environment variables, manual restart required
kubectl rollout restart deployment <deployment-name> -n <namespace>
```

### Forcing ConfigMap/Secret Reload

```bash
# Restart pods to reload volume-mounted configs
kubectl delete pod <pod-name> -n <namespace>

# Rolling restart for deployments
kubectl rollout restart deployment <deployment-name> -n <namespace>

# Use annotations to force pod recreation
kubectl patch deployment <deployment-name> -p '{"spec":{"template":{"metadata":{"annotations":{"configmap.reloader.stakater.com/reload":"'<configmap-name>'"}}}}}' -n <namespace>
```

---

## Debugging and Testing

### Manual Testing

```bash
# Create test pod to verify ConfigMap/Secret
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-test
  namespace: <namespace>
spec:
  containers:
  - name: test
    image: busybox
    command: ['sleep', '3600']
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
    - name: secret-volume
      mountPath: /etc/secret
    env:
    - name: CONFIG_VALUE
      valueFrom:
        configMapKeyRef:
          name: <configmap-name>
          key: <key>
    - name: SECRET_VALUE
      valueFrom:
        secretKeyRef:
          name: <secret-name>
          key: <key>
  volumes:
  - name: config-volume
    configMap:
      name: <configmap-name>
  - name: secret-volume
    secret:
      secretName: <secret-name>
EOF

# Test the configuration
kubectl exec -it config-test -n <namespace> -- ls -la /etc/config/
kubectl exec -it config-test -n <namespace> -- ls -la /etc/secret/
kubectl exec -it config-test -n <namespace> -- env | grep -E "CONFIG_VALUE|SECRET_VALUE"
```

### Validation Scripts

```bash
# Validate ConfigMap structure
kubectl get configmap <configmap-name> -n <namespace> -o json | jq '.data | keys[]'

# Validate Secret structure
kubectl get secret <secret-name> -n <namespace> -o json | jq '.data | keys[]'

# Check for common issues
kubectl get configmap <configmap-name> -n <namespace> -o yaml | grep -E "^\s*[^:]+:\s*$"  # Empty values
```

---

## Quick Reference Commands

```bash
# ConfigMap management
kubectl create configmap <name> --from-literal=<key>=<value> -n <namespace>
kubectl create configmap <name> --from-file=<file-path> -n <namespace>
kubectl create configmap <name> --from-env-file=<env-file> -n <namespace>
kubectl get configmaps -n <namespace>
kubectl describe configmap <name> -n <namespace>
kubectl delete configmap <name> -n <namespace>

# Secret management
kubectl create secret generic <name> --from-literal=<key>=<value> -n <namespace>
kubectl create secret generic <name> --from-file=<file-path> -n <namespace>
kubectl create secret tls <name> --cert=<cert-file> --key=<key-file> -n <namespace>
kubectl create secret docker-registry <name> --docker-server=<server> --docker-username=<user> --docker-password=<pass> -n <namespace>
kubectl get secrets -n <namespace>
kubectl describe secret <name> -n <namespace>

# Data inspection
kubectl get configmap <name> -n <namespace> -o yaml
kubectl get secret <name> -n <namespace> -o yaml
kubectl get configmap <name> -n <namespace> -o jsonpath='{.data.<key>}'
kubectl get secret <name> -n <namespace> -o jsonpath='{.data.<key>}' | base64 -d

# Volume mount testing
kubectl exec -it <pod-name> -n <namespace> -- ls -la /path/to/mount/
kubectl exec -it <pod-name> -n <namespace> -- cat /path/to/mount/<file>

# Environment variable testing
kubectl exec -it <pod-name> -n <namespace> -- env | grep <VAR_NAME>
kubectl exec -it <pod-name> -n <namespace> -- echo $<VAR_NAME>

# Updates and reloads
kubectl rollout restart deployment <deployment-name> -n <namespace>
kubectl delete pod <pod-name> -n <namespace>
```

## Best Practices

1. **Immutable ConfigMaps**: Use immutable ConfigMaps for better performance
2. **Separate Concerns**: Use separate ConfigMaps for different configuration aspects
3. **Naming Conventions**: Use consistent naming for keys and ConfigMap/Secret names
4. **Security**: Never store secrets in ConfigMaps, use Secrets instead
5. **Size Limits**: Keep ConfigMaps under 1MB, split large configs
6. **Documentation**: Document configuration keys and their purposes
7. **Validation**: Validate configuration syntax before creating ConfigMaps
8. **Automation**: Use tools like Helm or Kustomize for configuration management

## Common Patterns

### Application Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    server.port=8080
    database.url=jdbc:postgresql://db:5432/myapp
    logging.level=INFO
  nginx.conf: |
    server {
        listen 80;
        location / {
            proxy_pass http://backend:8080;
        }
    }
```

### Environment-Specific Configs

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
data:
  ENVIRONMENT: "production"
  LOG_LEVEL: "warn"
  FEATURE_FLAGS: "feature1=true,feature2=false"
```

### Mixed Configuration

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: url
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: log-level
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
      - key: app.properties
        path: application.properties
```

---

## Hands-On Testing Scenarios

Practice these scenarios in your Minikube or kind cluster to gain practical ConfigMap and Secret troubleshooting experience.

### Scenario 1: ConfigMap Not Found

**Setup the Problem:**

```bash
# Create pod referencing non-existent ConfigMap
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: missing-configmap-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
    env:
    - name: CONFIG_VALUE
      valueFrom:
        configMapKeyRef:
          name: nonexistent-config
          key: somekey
EOF
```

**Troubleshoot:**

```bash
# Check pod status
kubectl get pods
kubectl describe pod missing-configmap-pod

# Check if ConfigMap exists
kubectl get configmaps
```

**Fix:**

```bash
# Create the required ConfigMap
kubectl create configmap nonexistent-config --from-literal=somekey=somevalue

# Delete and recreate pod (environment variables are set at container creation)
kubectl delete pod missing-configmap-pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: missing-configmap-pod-fixed
spec:
  containers:
  - name: app
    image: nginx:alpine
    env:
    - name: CONFIG_VALUE
      valueFrom:
        configMapKeyRef:
          name: nonexistent-config
          key: somekey
EOF

# Verify fix
kubectl exec missing-configmap-pod-fixed -- env | grep CONFIG_VALUE
```

### Scenario 2: Secret Base64 Encoding Issues

**Setup the Problem:**

```bash
# Create Secret with incorrectly encoded data
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: encoding-issue-secret
type: Opaque
data:
  username: plaintext-not-encoded  # This should be base64 encoded
  password: cGFzc3dvcmQ=  # This is correctly encoded
EOF
```

**Troubleshoot:**

```bash
# Check Secret
kubectl get secret encoding-issue-secret -o yaml

# Try to decode the values
kubectl get secret encoding-issue-secret -o jsonpath='{.data.username}' | base64 -d || echo "Decoding failed"
kubectl get secret encoding-issue-secret -o jsonpath='{.data.password}' | base64 -d
```

**Fix:**

```bash
# Fix the encoding
kubectl patch secret encoding-issue-secret -p '{"data":{"username":"'$(echo -n "admin" | base64)'"}}'

# Verify fix
kubectl get secret encoding-issue-secret -o jsonpath='{.data.username}' | base64 -d
```

### Scenario 3: Volume Mount Permission Issues

**Setup the Problem:**

```bash
# Create ConfigMap and Secret
kubectl create configmap app-config --from-literal=app.conf="server_name=localhost"
kubectl create secret generic app-secret --from-literal=api-key="secret123"

# Create pod with restrictive security context
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: permission-test-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 1000
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
    - name: secret-volume
      mountPath: /etc/secret
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      defaultMode: 0644
  - name: secret-volume
    secret:
      secretName: app-secret
      defaultMode: 0600  # Too restrictive for the user
EOF
```

**Troubleshoot:**

```bash
# Check pod status
kubectl get pods
kubectl describe pod permission-test-pod

# Check file permissions
kubectl exec permission-test-pod -- ls -la /etc/config/
kubectl exec permission-test-pod -- ls -la /etc/secret/

# Try to read files
kubectl exec permission-test-pod -- cat /etc/config/app.conf
kubectl exec permission-test-pod -- cat /etc/secret/api-key
```

**Fix:**

```bash
# Fix permissions by adjusting defaultMode
kubectl patch pod permission-test-pod -p '{"spec":{"volumes":[{"name":"secret-volume","secret":{"secretName":"app-secret","defaultMode":432}}]}}'

# Or recreate pod with correct permissions
kubectl delete pod permission-test-pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: permission-test-pod-fixed
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 1000
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
    - name: secret-volume
      mountPath: /etc/secret
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      defaultMode: 0644
  - name: secret-volume
    secret:
      secretName: app-secret
      defaultMode: 0644  # Readable by user and group
EOF
```

### Scenario 4: ConfigMap Update Not Reflected

**Setup the Problem:**

```bash
# Create ConfigMap and deployment
kubectl create configmap update-test-config --from-literal=message="original message"

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-update-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-update-test
  template:
    metadata:
      labels:
        app: config-update-test
    spec:
      containers:
      - name: app
        image: busybox:latest
        command: ["sh", "-c", "while true; do echo \$MESSAGE; sleep 10; done"]
        env:
        - name: MESSAGE
          valueFrom:
            configMapKeyRef:
              name: update-test-config
              key: message
EOF

# Wait for pod to start
kubectl wait --for=condition=Ready pod -l app=config-update-test --timeout=60s

# Check initial message
kubectl logs -l app=config-update-test --tail=3

# Update ConfigMap
kubectl patch configmap update-test-config -p '{"data":{"message":"updated message"}}'

# Check if change is reflected
sleep 15
kubectl logs -l app=config-update-test --tail=3
```

**Troubleshoot:**

```bash
# Check ConfigMap
kubectl get configmap update-test-config -o yaml

# Check current environment variable in pod
kubectl exec deployment/config-update-test -- env | grep MESSAGE
```

**Fix:**

```bash
# For environment variables, need to restart deployment
kubectl rollout restart deployment config-update-test

# Wait for rollout
kubectl rollout status deployment config-update-test

# Check updated message
kubectl logs -l app=config-update-test --tail=3
```

### Scenario 5: Secret in Different Namespace

**Setup the Problem:**

```bash
# Create secret in different namespace
kubectl create namespace secret-namespace
kubectl create secret generic cross-namespace-secret \
  --from-literal=token="secret-token" \
  -n secret-namespace

# Try to use secret from default namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cross-namespace-test
spec:
  containers:
  - name: app
    image: nginx:alpine
    env:
    - name: SECRET_TOKEN
      valueFrom:
        secretKeyRef:
          name: cross-namespace-secret
          key: token
EOF
```

**Troubleshoot:**

```bash
# Check pod status
kubectl get pods
kubectl describe pod cross-namespace-test

# Check if secret exists in current namespace
kubectl get secrets
kubectl get secrets -n secret-namespace
```

**Fix:**

```bash
# Copy secret to current namespace
kubectl get secret cross-namespace-secret -n secret-namespace -o yaml | \
  sed 's/namespace: secret-namespace/namespace: default/' | \
  kubectl apply -f -

# Or create secret in correct namespace
kubectl delete pod cross-namespace-test
kubectl create secret generic cross-namespace-secret \
  --from-literal=token="secret-token"

# Recreate pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: cross-namespace-test-fixed
spec:
  containers:
  - name: app
    image: nginx:alpine
    env:
    - name: SECRET_TOKEN
      valueFrom:
        secretKeyRef:
          name: cross-namespace-secret
          key: token
EOF
```

### Scenario 6: SubPath Mount Issues

**Setup the Problem:**

```bash
# Create ConfigMap with multiple files
kubectl create configmap multifile-config \
  --from-literal=app.conf="server_name=localhost" \
  --from-literal=nginx.conf="worker_processes=1" \
  --from-literal=mime.types="text/plain txt"

# Create pod with subPath that conflicts with existing file
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: subpath-test-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/nginx.conf  # This conflicts with existing nginx.conf
      subPath: nginx.conf
    - name: config-volume
      mountPath: /etc/app/app.conf
      subPath: app.conf
  volumes:
  - name: config-volume
    configMap:
      name: multifile-config
EOF
```

**Troubleshoot:**

```bash
# Check pod status
kubectl describe pod subpath-test-pod

# Check if files are mounted correctly
kubectl exec subpath-test-pod -- ls -la /etc/nginx/
kubectl exec subpath-test-pod -- cat /etc/nginx/nginx.conf
kubectl exec subpath-test-pod -- ls -la /etc/app/
```

**Fix:**

```bash
# Mount to a different location
kubectl delete pod subpath-test-pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: subpath-test-pod-fixed
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: config-volume
      mountPath: /etc/custom/nginx.conf
      subPath: nginx.conf
    - name: config-volume
      mountPath: /etc/custom/app.conf
      subPath: app.conf
  volumes:
  - name: config-volume
    configMap:
      name: multifile-config
EOF
```

### Complete Testing Script

**Save this as `test-configmaps-secrets.sh`:**

```bash
#!/bin/bash

echo "=== Kubernetes ConfigMap and Secret Troubleshooting Scenarios ==="

# Scenario 1: Missing ConfigMap
echo "Creating pod with missing ConfigMap..."
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: missing-cm-test
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    env:
    - name: CONFIG_VAL
      valueFrom:
        configMapKeyRef:
          name: missing-configmap
          key: value
EOF

echo "Check: kubectl describe pod missing-cm-test"
echo "Fix: kubectl create configmap missing-configmap --from-literal=value=test"
echo ""

# Scenario 2: Encoding issues
echo "Creating secret with encoding issues..."
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: encoding-test
data:
  good-key: dGVzdA==  # base64 encoded "test"
  bad-key: not-encoded  # not base64 encoded
EOF

echo "Check: kubectl get secret encoding-test -o yaml"
echo "Decode: kubectl get secret encoding-test -o jsonpath='{.data.bad-key}' | base64 -d"
echo ""

echo "Cleanup: kubectl delete pod missing-cm-test; kubectl delete secret encoding-test"
```

### Practice Exercise

1. **Set up your environment:**
   ```bash
   minikube start
   # or
   kind create cluster
   ```

2. **Work through each scenario:**
   - Create the problematic configuration
   - Use diagnostic commands to identify issues
   - Apply fixes and verify solutions

3. **Test different mounting options:**
   ```bash
   # Test volume mounts vs environment variables
   # Test subPath vs full volume mounts
   # Test different defaultMode values
   ```

4. **Monitor configuration changes:**
   ```bash
   # Watch ConfigMap changes
   kubectl get configmaps -w
   
   # Monitor pod restart behavior
   kubectl get pods -w
   ```

5. **Clean up:**
   ```bash
   kubectl delete pods,configmaps,secrets --all
   ```

### Advanced Scenarios

**Immutable ConfigMaps:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: immutable-config
data:
  setting: "value1"
immutable: true
EOF

# Try to update (should fail)
kubectl patch configmap immutable-config -p '{"data":{"setting":"value2"}}'
```

**Projected volumes:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: projected-volume-test
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: projected-config
      mountPath: /etc/config
  volumes:
  - name: projected-config
    projected:
      sources:
      - configMap:
          name: app-config
      - secret:
          name: app-secret
EOF
```

Remember: ConfigMaps and Secrets are fundamental to Kubernetes configuration management, and proper troubleshooting requires understanding both their creation and consumption patterns.
