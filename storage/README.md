# Storage Troubleshooting Guide

Guide for troubleshooting Kubernetes PersistentVolume and PersistentVolumeClaim issues.

## Common Storage Issues

### PVC Stuck in Pending

```bash
# Check PVC status
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>

# Check available PVs
kubectl get pv

# Check storage class
kubectl get storageclass
kubectl describe storageclass <storage-class-name>

# Check provisioner logs
kubectl logs -n kube-system <storage-provisioner-pod>
```

### Pod Cannot Mount Volume

```bash
# Check pod events
kubectl describe pod <pod-name> -n <namespace>

# Check PV status
kubectl describe pv <pv-name>

# Check node where pod is scheduled
kubectl get pod <pod-name> -n <namespace> -o wide
kubectl describe node <node-name>

# Check volume plugin logs
kubectl logs -n kube-system <csi-driver-pod>
```

### Storage Performance Issues

```bash
# Check disk I/O on nodes
kubectl exec -it <pod-name> -n <namespace> -- iostat -x 1

# Check volume usage
kubectl exec -it <pod-name> -n <namespace> -- df -h

# Check for storage errors
kubectl get events --all-namespaces | grep -i "volume\|storage\|mount"
```

## Quick Commands

```bash
# PVC management
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>
kubectl delete pvc <pvc-name> -n <namespace>

# PV management
kubectl get pv
kubectl describe pv <pv-name>
kubectl delete pv <pv-name>

# Storage class
kubectl get storageclass
kubectl describe storageclass <sc-name>

# Volume inspection
kubectl exec -it <pod-name> -n <namespace> -- df -h
kubectl exec -it <pod-name> -n <namespace> -- ls -la /path/to/volume
```

---

## Hands-On Testing Scenarios

Practice these scenarios in your Minikube or kind cluster to gain practical Storage troubleshooting experience.

### Scenario 1: PVC Pending Due to No Available PV

**Setup the Problem:**

```bash
# Create PVC without corresponding PV or StorageClass
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: orphaned-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: "nonexistent-storage-class"
EOF
```

**Troubleshoot:**

```bash
# Check PVC status
kubectl get pvc
kubectl describe pvc orphaned-pvc

# Check available storage classes
kubectl get storageclass

# Check for available PVs
kubectl get pv

# Check events for provisioning failures
kubectl get events --sort-by=.metadata.creationTimestamp | grep orphaned-pvc
```

**Fix:**

```bash
# Option 1: Create a matching PV manually
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/manual-pv
EOF

# Update PVC to not specify storage class (so it uses manual PV)
kubectl patch pvc orphaned-pvc -p '{"spec":{"storageClassName":""}}'

# Option 2: Use default storage class
kubectl patch pvc orphaned-pvc -p '{"spec":{"storageClassName":"standard"}}' # for minikube

# Verify PVC is bound
kubectl get pvc orphaned-pvc
```

### Scenario 2: Pod Cannot Mount Volume

**Setup the Problem:**

```bash
# Create PVC with correct storage class
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mount-test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: "standard"  # Use default for minikube/kind
EOF

# Wait for PVC to be bound
kubectl wait --for=condition=Bound pvc/mount-test-pvc --timeout=60s

# Create pod with mount issues (wrong path permissions)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: mount-issue-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 2000
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: data-volume
      mountPath: /usr/share/nginx/html  # nginx runs as user 101, may have permission issues
    securityContext:
      runAsNonRoot: true
      allowPrivilegeEscalation: false
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: mount-test-pvc
EOF
```

**Troubleshoot:**

```bash
# Check pod status
kubectl get pods
kubectl describe pod mount-issue-pod

# Check volume mounts
kubectl exec mount-issue-pod -- df -h
kubectl exec mount-issue-pod -- ls -la /usr/share/nginx/

# Check permission issues
kubectl exec mount-issue-pod -- id
kubectl exec mount-issue-pod -- ls -la /usr/share/nginx/html/
```

**Fix:**

```bash
# Fix by adjusting security context
kubectl delete pod mount-issue-pod

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: mount-fixed-pod
spec:
  securityContext:
    runAsUser: 101  # nginx user
    runAsGroup: 101
    fsGroup: 101
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: data-volume
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: mount-test-pvc
EOF

# Verify fix
kubectl wait --for=condition=Ready pod/mount-fixed-pod --timeout=60s
kubectl exec mount-fixed-pod -- ls -la /usr/share/nginx/html/
```

### Scenario 3: Volume Size Expansion Issues

**Setup the Problem:**

```bash
# Create PVC with initial size
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: expansion-test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
  storageClassName: "standard"
EOF

kubectl wait --for=condition=Bound pvc/expansion-test-pvc --timeout=60s

# Create pod using the PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: expansion-test-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: expansion-test-pvc
EOF

kubectl wait --for=condition=Ready pod/expansion-test-pod --timeout=60s

# Check initial size
kubectl exec expansion-test-pod -- df -h /data

# Try to expand volume (may fail depending on storage class)
kubectl patch pvc expansion-test-pvc -p '{"spec":{"resources":{"requests":{"storage":"200Mi"}}}}'
```

**Troubleshoot:**

```bash
# Check if storage class supports expansion
kubectl get storageclass standard -o yaml | grep allowVolumeExpansion

# Check PVC status for expansion
kubectl get pvc expansion-test-pvc
kubectl describe pvc expansion-test-pvc

# Check events for expansion issues
kubectl get events --sort-by=.metadata.creationTimestamp | grep expansion-test-pvc

# Check if filesystem expansion is needed
kubectl exec expansion-test-pod -- df -h /data
```

**Fix:**

```bash
# For minikube/kind, create storage class that supports expansion
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-storage
provisioner: k8s.io/minikube-hostpath  # for minikube
allowVolumeExpansion: true
parameters:
  type: pd-standard
EOF

# Create new PVC with expandable storage class
kubectl delete pod expansion-test-pod
kubectl delete pvc expansion-test-pvc

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: expandable-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
  storageClassName: "expandable-storage"
EOF

# Test expansion with new PVC
kubectl wait --for=condition=Bound pvc/expandable-pvc --timeout=60s
kubectl patch pvc expandable-pvc -p '{"spec":{"resources":{"requests":{"storage":"200Mi"}}}}'
```

### Scenario 4: Multiple Pods Cannot Share Volume

**Setup the Problem:**

```bash
# Create ReadWriteOnce PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rwo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
  storageClassName: "standard"
EOF

kubectl wait --for=condition=Bound pvc/rwo-pvc --timeout=60s

# Create first pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    persistentVolumeClaim:
      claimName: rwo-pvc
EOF

kubectl wait --for=condition=Ready pod/pod1 --timeout=60s

# Try to create second pod using same PVC (will fail)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pod2
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    persistentVolumeClaim:
      claimName: rwo-pvc
EOF
```

**Troubleshoot:**

```bash
# Check pod statuses
kubectl get pods
kubectl describe pod pod2

# Check PVC access modes
kubectl get pvc rwo-pvc -o yaml | grep accessModes -A 3

# Check available storage classes and their capabilities
kubectl get storageclass -o wide

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp | grep pod2
```

**Fix:**

```bash
# Solution 1: Use ReadWriteMany if supported by storage class
# (Note: hostPath doesn't support RWX, so create a different example)

# Solution 2: Use shared volume approach
kubectl delete pod pod1 pod2
kubectl delete pvc rwo-pvc

# Create deployment that scales pods on same node
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
  storageClassName: "standard"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shared-volume-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shared-app
  template:
    metadata:
      labels:
        app: shared-app
    spec:
      containers:
      - name: app
        image: busybox
        command: ["sleep", "3600"]
        volumeMounts:
        - name: shared-data
          mountPath: /data
      volumes:
      - name: shared-data
        persistentVolumeClaim:
          claimName: shared-pvc
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - shared-app
            topologyKey: kubernetes.io/hostname
EOF
```

### Scenario 5: Storage Class Performance Issues

**Setup the Problem:**

```bash
# Create custom storage class with specific parameters
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow-storage
provisioner: k8s.io/minikube-hostpath
parameters:
  type: slow-disk
reclaimPolicy: Retain
volumeBindingMode: Immediate
EOF

# Create PVC and pod for performance testing
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: perf-test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: "slow-storage"
---
apiVersion: v1
kind: Pod
metadata:
  name: perf-test-pod
spec:
  containers:
  - name: perf-tester
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: test-volume
      mountPath: /test
  volumes:
  - name: test-volume
    persistentVolumeClaim:
      claimName: perf-test-pvc
EOF

kubectl wait --for=condition=Ready pod/perf-test-pod --timeout=60s
```

**Troubleshoot:**

```bash
# Test write performance
kubectl exec perf-test-pod -- dd if=/dev/zero of=/test/testfile bs=1M count=100

# Test read performance
kubectl exec perf-test-pod -- dd if=/test/testfile of=/dev/null bs=1M

# Check storage class parameters
kubectl describe storageclass slow-storage

# Monitor I/O metrics (if available)
kubectl top pod perf-test-pod

# Check underlying storage
kubectl get pv -o wide
```

**Fix:**

```bash
# Create optimized storage class
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: k8s.io/minikube-hostpath
parameters:
  type: fast-ssd
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF

# Migrate to better performing storage
kubectl delete pod perf-test-pod
kubectl delete pvc perf-test-pvc

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-perf-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: "fast-storage"
---
apiVersion: v1
kind: Pod
metadata:
  name: fast-perf-pod
spec:
  containers:
  - name: perf-tester
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: test-volume
      mountPath: /test
  volumes:
  - name: test-volume
    persistentVolumeClaim:
      claimName: fast-perf-pvc
EOF

# Test improved performance
kubectl wait --for=condition=Ready pod/fast-perf-pod --timeout=60s
kubectl exec fast-perf-pod -- dd if=/dev/zero of=/test/testfile bs=1M count=100
```

### Scenario 6: Volume Cleanup and Reclaim Issues

**Setup the Problem:**

```bash
# Create PV with Retain policy
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: retain-pv
spec:
  capacity:
    storage: 500Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/retain-data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: retain-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: ""
  volumeName: retain-pv
EOF

kubectl wait --for=condition=Bound pvc/retain-pvc --timeout=60s

# Create pod and write data
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: data-writer-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "echo 'important data' > /data/important.txt && sleep 3600"]
    volumeMounts:
    - name: data-volume
      mountPath: /data
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: retain-pvc
EOF

kubectl wait --for=condition=Ready pod/data-writer-pod --timeout=60s

# Delete PVC (PV should remain due to Retain policy)
kubectl delete pod data-writer-pod
kubectl delete pvc retain-pvc
```

**Troubleshoot:**

```bash
# Check PV status after PVC deletion
kubectl get pv retain-pv

# PV should be in "Released" state
kubectl describe pv retain-pv

# Check if data still exists (for hostPath)
if command -v minikube &> /dev/null; then
    minikube ssh "ls -la /tmp/retain-data/"
fi
```

**Fix:**

```bash
# To reuse the PV, need to remove claimRef
kubectl patch pv retain-pv -p '{"spec":{"claimRef":null}}'

# Verify PV is Available again
kubectl get pv retain-pv

# Create new PVC to claim the PV
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: reuse-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: ""
  volumeName: retain-pv
EOF

# Verify data persistence
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: data-reader-pod
spec:
  containers:
  - name: reader
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: data-volume
      mountPath: /data
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: reuse-pvc
EOF

kubectl wait --for=condition=Ready pod/data-reader-pod --timeout=60s
kubectl exec data-reader-pod -- cat /data/important.txt
```

### Complete Testing Script

**Save this as `test-storage.sh`:**

```bash
#!/bin/bash

echo "=== Kubernetes Storage Troubleshooting Scenarios ==="

echo "=== Scenario 1: Basic PVC Creation ==="
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: basic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
  storageClassName: "standard"
EOF

echo "Checking PVC status..."
kubectl get pvc basic-pvc
kubectl describe pvc basic-pvc

echo ""
echo "=== Scenario 2: Pod with Volume Mount ==="
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: volume-test-pod
spec:
  containers:
  - name: test-container
    image: busybox
    command: ["sleep", "300"]
    volumeMounts:
    - name: test-volume
      mountPath: /test-data
  volumes:
  - name: test-volume
    persistentVolumeClaim:
      claimName: basic-pvc
EOF

echo "Waiting for pod to be ready..."
kubectl wait --for=condition=Ready pod/volume-test-pod --timeout=60s

echo "Testing volume mount..."
kubectl exec volume-test-pod -- df -h /test-data
kubectl exec volume-test-pod -- sh -c "echo 'test data' > /test-data/test.txt"
kubectl exec volume-test-pod -- cat /test-data/test.txt

echo ""
echo "=== Storage Information ==="
echo "Available storage classes:"
kubectl get storageclass

echo ""
echo "Current PVs:"
kubectl get pv

echo ""
echo "Current PVCs:"
kubectl get pvc

echo ""
echo "=== Cleanup Commands ==="
echo "kubectl delete pod volume-test-pod"
echo "kubectl delete pvc basic-pvc"
```

### Practice Exercise

1. **Set up storage environment:**
   ```bash
   # For Minikube, enable default storage
   minikube addons enable default-storageclass
   minikube addons enable storage-provisioner
   
   # Check available storage classes
   kubectl get storageclass
   ```

2. **Test different access modes:**
   ```bash
   # Create PVCs with different access modes
   # Test ReadWriteOnce vs ReadOnlyMany behavior
   # Understand when each mode is appropriate
   ```

3. **Practice volume operations:**
   ```bash
   # Create, bind, use, and delete PVCs
   # Test volume expansion (if supported)
   # Practice data persistence scenarios
   ```

4. **Monitor storage usage:**
   ```bash
   # Check storage consumption
   kubectl top nodes
   kubectl describe node <node-name> | grep -A 10 "Allocated resources"
   ```

5. **Clean up:**
   ```bash
   kubectl delete pods,pvc --all
   kubectl get pv  # Check for any remaining PVs that need manual cleanup
   ```

### Advanced Scenarios

**StatefulSet with Volume Claims:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: storage-test-sts
spec:
  serviceName: storage-test
  replicas: 3
  selector:
    matchLabels:
      app: storage-test
  template:
    metadata:
      labels:
        app: storage-test
    spec:
      containers:
      - name: app
        image: busybox
        command: ["sleep", "3600"]
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 100Mi
      storageClassName: "standard"
EOF
```

**Snapshot and Restore (if supported):**

```bash
# Check if volume snapshots are supported
kubectl get volumesnapshotclass

# Create volume snapshot
cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: test-snapshot
spec:
  source:
    persistentVolumeClaimName: basic-pvc
EOF
```

Remember: Storage troubleshooting often involves understanding the underlying storage provider's behavior and limitations. Always test backup and restore procedures in non-production environments.
