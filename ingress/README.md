# Ingress Troubleshooting Guide

Comprehensive guide for troubleshooting Kubernetes Ingress routing and SSL/TLS issues.

## Table of Contents

1. [Ingress Controller Issues](#ingress-controller-issues)
2. [Ingress Not Routing Traffic](#ingress-not-routing-traffic)
3. [SSL/TLS Certificate Issues](#ssltls-certificate-issues)
4. [DNS and Domain Issues](#dns-and-domain-issues)
5. [Backend Service Problems](#backend-service-problems)
6. [Path-Based Routing Issues](#path-based-routing-issues)
7. [Host-Based Routing Issues](#host-based-routing-issues)
8. [Load Balancer Integration](#load-balancer-integration)
9. [Quick Reference Commands](#quick-reference-commands)

---

## Ingress Controller Issues

### Ingress Controller Not Running

#### Diagnosis Steps

```bash
# Check ingress controller pods
kubectl get pods -n ingress-nginx  # for nginx ingress
kubectl get pods -n kube-system | grep ingress

# Check ingress controller deployment
kubectl get deployment -n ingress-nginx
kubectl describe deployment ingress-nginx-controller -n ingress-nginx

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

#### Common Issues

**No Ingress Controller Installed**:

```bash
# Install nginx ingress controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml

# Verify installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

**Controller Not Ready**:

```bash
# Check controller readiness
kubectl describe pod <ingress-controller-pod> -n ingress-nginx

# Check controller service
kubectl get svc ingress-nginx-controller -n ingress-nginx
```

### Ingress Class Configuration

```bash
# Check available ingress classes
kubectl get ingressclass

# Set default ingress class
kubectl annotate ingressclass nginx ingressclass.kubernetes.io/is-default-class=true

# Check ingress resource ingress class
kubectl get ingress <ingress-name> -n <namespace> -o yaml | grep ingressClassName
```

---

## Ingress Not Routing Traffic

### Diagnosis Steps

```bash
# Check ingress resource status
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>

# Check ingress controller configuration
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx | grep <ingress-name>

# Test backend service directly
kubectl get svc <backend-service> -n <namespace>
kubectl describe svc <backend-service> -n <namespace>
```

### Common Routing Issues

#### 1. Incorrect Ingress Configuration

```bash
# Check ingress rules
kubectl get ingress <ingress-name> -n <namespace> -o yaml

# Verify host and path configuration
kubectl describe ingress <ingress-name> -n <namespace>
```

**Example correct ingress**:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: example-service
            port:
              number: 80
```

#### 2. Service Backend Issues

```bash
# Check if backend service exists
kubectl get svc <service-name> -n <namespace>

# Check service endpoints
kubectl get endpoints <service-name> -n <namespace>

# Test service directly
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -qO- <service-name>:<port>
```

#### 3. Ingress Class Mismatch

```bash
# Check ingress class in resource
kubectl get ingress <ingress-name> -n <namespace> -o yaml | grep ingressClassName

# Check available ingress classes
kubectl get ingressclass

# Update ingress class if needed
kubectl patch ingress <ingress-name> -n <namespace> -p '{"spec":{"ingressClassName":"nginx"}}'
```

---

## SSL/TLS Certificate Issues

### Certificate Not Loading

#### Diagnosis Steps

```bash
# Check TLS configuration in ingress
kubectl get ingress <ingress-name> -n <namespace> -o yaml | grep -A 10 tls

# Check if certificate secret exists
kubectl get secrets -n <namespace>
kubectl describe secret <tls-secret> -n <namespace>

# Verify certificate content
kubectl get secret <tls-secret> -n <namespace> -o yaml
```

### Certificate Manager Issues

#### cert-manager Troubleshooting

```bash
# Check cert-manager pods
kubectl get pods -n cert-manager

# Check certificate status
kubectl get certificates -n <namespace>
kubectl describe certificate <cert-name> -n <namespace>

# Check certificate requests
kubectl get certificaterequests -n <namespace>
kubectl describe certificaterequest <cert-request> -n <namespace>

# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager
kubectl logs -n cert-manager -l app=cainjector
kubectl logs -n cert-manager -l app=webhook
```

#### Let's Encrypt Issues

```bash
# Check ACME challenge status
kubectl get challenges -n <namespace>
kubectl describe challenge <challenge-name> -n <namespace>

# Check HTTP01 challenge accessibility
curl -v http://<domain>/.well-known/acme-challenge/<token>

# Check DNS01 challenge (if using DNS validation)
dig TXT _acme-challenge.<domain>
```

### Manual Certificate Management

```bash
# Create TLS secret manually
kubectl create secret tls <secret-name> \
  --cert=<path-to-cert-file> \
  --key=<path-to-key-file> \
  -n <namespace>

# Verify certificate
openssl x509 -in <cert-file> -text -noout
openssl x509 -in <cert-file> -noout -dates

# Check certificate chain
openssl s_client -connect <domain>:443 -showcerts
```

---

## DNS and Domain Issues

### Domain Not Resolving

```bash
# Check DNS resolution
nslookup <domain>
dig <domain>

# Check if domain points to correct IP
kubectl get svc ingress-nginx-controller -n ingress-nginx -o wide

# Test DNS from different locations
# Use online DNS checking tools
```

### Subdomain Issues

```bash
# Check wildcard certificate
kubectl get secret <tls-secret> -n <namespace> -o yaml | grep -A 5 tls.crt | base64 -d | openssl x509 -text -noout | grep -A 2 "Subject Alternative Name"

# Test subdomain routing
curl -H "Host: subdomain.example.com" http://<ingress-ip>/
```

---

## Backend Service Problems

### Service Not Responding

```bash
# Check service health
kubectl get endpoints <service-name> -n <namespace>

# Test service directly
kubectl port-forward svc/<service-name> 8080:80 -n <namespace>
curl http://localhost:8080

# Check pod health
kubectl get pods -n <namespace> -l <service-selector>
kubectl describe pod <pod-name> -n <namespace>
```

### Service Port Mismatch

```bash
# Check service ports
kubectl get svc <service-name> -n <namespace> -o yaml | grep -A 5 ports

# Check ingress backend port
kubectl get ingress <ingress-name> -n <namespace> -o yaml | grep -A 5 port

# Verify container ports
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 5 ports
```

---

## Path-Based Routing Issues

### Path Not Matching

```bash
# Check path configuration
kubectl get ingress <ingress-name> -n <namespace> -o yaml | grep -A 10 paths

# Test different path types
# Prefix vs Exact matching
```

**Path Type Examples**:

```yaml
# Prefix matching (most common)
- path: /api
  pathType: Prefix
  
# Exact matching
- path: /api/v1/health
  pathType: Exact
  
# Implementation-specific (nginx uses regex)
- path: /api/(.*) 
  pathType: ImplementationSpecific
```

### URL Rewriting Issues

```bash
# Check nginx ingress annotations
kubectl get ingress <ingress-name> -n <namespace> -o yaml | grep -A 10 annotations

# Common nginx annotations for path handling
# nginx.ingress.kubernetes.io/rewrite-target: /
# nginx.ingress.kubernetes.io/use-regex: "true"
```

---

## Host-Based Routing Issues

### Multiple Hosts Not Working

```bash
# Check host rules in ingress
kubectl get ingress <ingress-name> -n <namespace> -o yaml | grep -A 20 rules

# Test each host individually
curl -H "Host: host1.example.com" http://<ingress-ip>/
curl -H "Host: host2.example.com" http://<ingress-ip>/
```

### Default Backend Issues

```bash
# Check if default backend is configured
kubectl get ingress <ingress-name> -n <namespace> -o yaml | grep -A 5 defaultBackend

# Test default backend
curl http://<ingress-ip>/nonexistent-path
```

---

## Load Balancer Integration

### External IP Not Assigned

```bash
# Check ingress controller service
kubectl get svc ingress-nginx-controller -n ingress-nginx

# Check load balancer provisioning
kubectl describe svc ingress-nginx-controller -n ingress-nginx

# Check cloud provider integration
kubectl logs -n kube-system -l component=cloud-controller-manager
```

### Load Balancer Health Checks

```bash
# Check health check endpoint
curl http://<external-ip>/healthz

# Check ingress controller health
kubectl get pods -n ingress-nginx
kubectl describe pod <ingress-controller-pod> -n ingress-nginx
```

---

## Advanced Troubleshooting

### Ingress Controller Configuration

```bash
# Check nginx configuration
kubectl exec -n ingress-nginx <ingress-controller-pod> -- cat /etc/nginx/nginx.conf

# Check ingress controller configmap
kubectl get configmap ingress-nginx-controller -n ingress-nginx -o yaml

# Reload nginx configuration
kubectl exec -n ingress-nginx <ingress-controller-pod> -- nginx -s reload
```

### Traffic Analysis

```bash
# Enable ingress controller debug logging
kubectl patch configmap ingress-nginx-controller -n ingress-nginx -p '{"data":{"error-log-level":"debug"}}'

# Check access logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -f

# Test with verbose curl
curl -v -H "Host: example.com" http://<ingress-ip>/path
```

### Performance Issues

```bash
# Check ingress controller resource usage
kubectl top pod -n ingress-nginx

# Check connection limits
kubectl get configmap ingress-nginx-controller -n ingress-nginx -o yaml | grep -E "worker-connections|max-worker-connections"

# Monitor active connections
kubectl exec -n ingress-nginx <ingress-controller-pod> -- nginx -T | grep worker_connections
```

---

## Quick Reference Commands

```bash
# Ingress management
kubectl create ingress <name> --rule="<host>/<path>=<service>:<port>" -n <namespace>
kubectl get ingress -n <namespace>
kubectl describe ingress <name> -n <namespace>
kubectl delete ingress <name> -n <namespace>

# TLS/SSL management
kubectl create secret tls <secret-name> --cert=<cert-file> --key=<key-file> -n <namespace>
kubectl get secrets -n <namespace>

# Ingress controller
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
kubectl get svc ingress-nginx-controller -n ingress-nginx

# Testing
curl -H "Host: <hostname>" http://<ingress-ip>/<path>
curl -v https://<hostname>/<path>

# cert-manager (if used)
kubectl get certificates -n <namespace>
kubectl describe certificate <cert-name> -n <namespace>
kubectl get certificaterequests -n <namespace>
kubectl get challenges -n <namespace>

# Configuration
kubectl get ingressclass
kubectl get configmap ingress-nginx-controller -n ingress-nginx -o yaml
```

## Best Practices

1. **Use Ingress Classes**: Specify ingress class for better control
2. **Implement Health Checks**: Ensure backend services are healthy
3. **Certificate Management**: Use cert-manager for automatic certificate renewal
4. **Path Planning**: Design URL paths carefully for scalability
5. **Security Headers**: Configure security headers in ingress controller
6. **Rate Limiting**: Implement rate limiting to protect backend services
7. **Monitoring**: Monitor ingress traffic and response times
8. **Documentation**: Document ingress rules and certificate requirements

## Common Ingress Patterns

### Multiple Services

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-service-ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
```

### TLS Termination

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - secure.example.com
    secretName: tls-secret
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### URL Rewriting

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api/v1(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 80
```

---

## Hands-On Testing Scenarios

Practice these scenarios in your Minikube or kind cluster to gain practical ingress troubleshooting experience.

### Prerequisites

```bash
# Enable ingress addon in minikube
minikube addons enable ingress

# Or install nginx ingress in kind
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for ingress controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### Scenario 1: Ingress Without Backend Service

**Setup the Problem:**

```bash
# Create ingress pointing to non-existent service
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: no-backend-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: test.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nonexistent-service
            port:
              number: 80
EOF
```

**Troubleshoot:**

```bash
# Check ingress status
kubectl get ingress no-backend-ingress
kubectl describe ingress no-backend-ingress

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Test access
curl -H "Host: test.local" http://$(minikube ip)/
```

**Fix:**

```bash
# Create the backend service and deployment
kubectl create deployment test-app --image=nginx:alpine
kubectl expose deployment test-app --port=80

# Update ingress to point to existing service
kubectl patch ingress no-backend-ingress -p '{"spec":{"rules":[{"host":"test.local","http":{"paths":[{"path":"/","pathType":"Prefix","backend":{"service":{"name":"test-app","port":{"number":80}}}}]}}]}}'
```

### Scenario 2: SSL/TLS Certificate Issues

**Setup the Problem:**

```bash
# Create deployment and service
kubectl create deployment ssl-app --image=nginx:alpine
kubectl expose deployment ssl-app --port=80

# Create invalid TLS secret
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=wrong-domain.com/O=wrong-domain.com"

kubectl create secret tls invalid-tls-secret --key tls.key --cert tls.crt

# Create ingress with TLS
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ssl-test-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - ssl-test.local
    secretName: invalid-tls-secret
  rules:
  - host: ssl-test.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ssl-app
            port:
              number: 80
EOF
```

**Troubleshoot:**

```bash
# Check certificate
kubectl describe secret invalid-tls-secret
kubectl get secret invalid-tls-secret -o yaml | grep tls.crt | cut -d: -f2 | base64 -d | openssl x509 -text -noout

# Test HTTPS access
curl -k -H "Host: ssl-test.local" https://$(minikube ip)/
```

**Fix:**

```bash
# Create correct certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls-fixed.key -out tls-fixed.crt \
  -subj "/CN=ssl-test.local/O=ssl-test.local"

kubectl create secret tls fixed-tls-secret --key tls-fixed.key --cert tls-fixed.crt

# Update ingress to use correct secret
kubectl patch ingress ssl-test-ingress -p '{"spec":{"tls":[{"hosts":["ssl-test.local"],"secretName":"fixed-tls-secret"}]}}'
```

### Scenario 3: Path-Based Routing Issues

**Setup the Problem:**

```bash
# Create two different apps
kubectl create deployment api-app --image=nginx:alpine
kubectl create deployment web-app --image=httpd:alpine

kubectl expose deployment api-app --port=80
kubectl expose deployment web-app --port=80

# Create ingress with wrong path configuration
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /api
        pathType: Exact  # Wrong pathType for prefix matching
        backend:
          service:
            name: api-app
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app
            port:
              number: 80
EOF
```

**Troubleshoot:**

```bash
# Test different paths
curl -H "Host: myapp.local" http://$(minikube ip)/
curl -H "Host: myapp.local" http://$(minikube ip)/api
curl -H "Host: myapp.local" http://$(minikube ip)/api/v1
```

**Fix:**

```bash
# Fix pathType to Prefix
kubectl patch ingress path-routing-ingress -p '{"spec":{"rules":[{"host":"myapp.local","http":{"paths":[{"path":"/api","pathType":"Prefix","backend":{"service":{"name":"api-app","port":{"number":80}}}},{"path":"/","pathType":"Prefix","backend":{"service":{"name":"web-app","port":{"number":80}}}}]}}]}}'
```

### Scenario 4: Host-Based Routing Issues

**Setup the Problem:**

```bash
# Create services
kubectl create deployment prod-app --image=nginx:alpine
kubectl create deployment staging-app --image=httpd:alpine

kubectl expose deployment prod-app --port=80
kubectl expose deployment staging-app --port=80

# Create ingress with host issues
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-routing-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: prod.myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prod-app
            port:
              number: 80
  - host: staging.myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: staging-app
            port:
              number: 80
EOF
```

**Test and Troubleshoot:**

```bash
# Test different hosts
curl -H "Host: prod.myapp.local" http://$(minikube ip)/
curl -H "Host: staging.myapp.local" http://$(minikube ip)/
curl -H "Host: unknown.myapp.local" http://$(minikube ip)/  # Should get 404

# Check ingress configuration
kubectl describe ingress host-routing-ingress
```

### Scenario 5: URL Rewriting Issues

**Setup the Problem:**

```bash
# Create API service
kubectl create deployment rewrite-app --image=nginx:alpine
kubectl expose deployment rewrite-app --port=80

# Create ingress with rewrite annotation
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /\$2  # Wrong regex group
spec:
  ingressClassName: nginx
  rules:
  - host: rewrite.local
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: rewrite-app
            port:
              number: 80
EOF
```

**Troubleshoot:**

```bash
# Test URL rewriting
curl -H "Host: rewrite.local" http://$(minikube ip)/api/test
curl -H "Host: rewrite.local" http://$(minikube ip)/api/

# Check nginx configuration
kubectl exec -n ingress-nginx <ingress-controller-pod> -- cat /etc/nginx/nginx.conf | grep -A 10 rewrite.local
```

**Fix:**

```bash
# Fix the rewrite annotation
kubectl annotate ingress rewrite-ingress nginx.ingress.kubernetes.io/rewrite-target=/\$2 --overwrite
```

### Scenario 6: Ingress Controller Issues

**Setup the Problem:**

```bash
# Scale down ingress controller to simulate failure
kubectl scale deployment ingress-nginx-controller --replicas=0 -n ingress-nginx

# Create test resources
kubectl create deployment controller-test --image=nginx:alpine
kubectl expose deployment controller-test --port=80

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: controller-test-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: controller-test.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: controller-test
            port:
              number: 80
EOF
```

**Troubleshoot:**

```bash
# Check ingress status
kubectl get ingress controller-test-ingress
kubectl describe ingress controller-test-ingress

# Check ingress controller
kubectl get pods -n ingress-nginx
kubectl get deployment ingress-nginx-controller -n ingress-nginx
```

**Fix:**

```bash
# Scale ingress controller back up
kubectl scale deployment ingress-nginx-controller --replicas=1 -n ingress-nginx

# Wait for controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### Complete Testing Script

**Save this as `test-ingress.sh`:**

```bash
#!/bin/bash

echo "=== Kubernetes Ingress Troubleshooting Scenarios ==="

# Prerequisites check
echo "Checking ingress controller..."
kubectl get pods -n ingress-nginx

echo "Creating test scenarios..."

# Scenario 1: Missing backend
kubectl create deployment backend-missing --image=nginx:alpine
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: missing-backend-test
spec:
  ingressClassName: nginx
  rules:
  - host: test1.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nonexistent-svc
            port:
              number: 80
EOF

# Scenario 2: Wrong service port
kubectl expose deployment backend-missing --port=80
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wrong-port-test
spec:
  ingressClassName: nginx
  rules:
  - host: test2.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-missing
            port:
              number: 8080  # Wrong port
EOF

echo ""
echo "Test these scenarios:"
echo "1. kubectl get ingress"
echo "2. kubectl describe ingress <ingress-name>"
echo "3. curl -H 'Host: test1.local' http://\$(minikube ip)/"
echo ""
echo "Cleanup: kubectl delete ingress,deployment,service --all"
```

### Practice Exercise

1. **Set up your environment:**
   ```bash
   minikube start
   minikube addons enable ingress
   ```

2. **Work through each scenario:**
   - Create the problematic configuration
   - Use diagnostic commands
   - Test with curl
   - Apply fixes and verify

3. **Test with real domains (optional):**
   ```bash
   # Add entries to /etc/hosts
   echo "$(minikube ip) test.local" | sudo tee -a /etc/hosts
   # Then test with: curl http://test.local/
   ```

4. **Advanced testing:**
   ```bash
   # Test with multiple ingress controllers
   # Test with different ingress classes
   # Test with annotations
   ```

5. **Clean up:**
   ```bash
   kubectl delete ingress,deployment,service --all
   minikube delete
   ```

### Monitoring Commands

```bash
# Watch ingress changes
kubectl get ingress -w

# Monitor ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -f

# Check ingress controller metrics
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller-metrics 9113:10254
curl http://localhost:9113/metrics
```

Remember: Ingress troubleshooting often involves multiple layers including DNS, certificates, load balancers, and backend services.
