# Kubernetes Troubleshooting Guide

A comprehensive guide for troubleshooting common Kubernetes issues across different resources. This repository is organized by resource type for easy navigation and reference.

## Repository Structure

```
k8s-troubleshooting/
├── README.md                    # This file - overview and general commands
├── pods/                        # Pod troubleshooting guide
├── deployments/                 # Deployment troubleshooting guide
├── services/                    # Service troubleshooting guide
├── ingress/                     # Ingress troubleshooting guide
├── configmaps-secrets/          # ConfigMap and Secret troubleshooting
├── nodes/                       # Node troubleshooting guide
├── namespaces/                  # Namespace troubleshooting guide
├── storage/                     # PV and PVC troubleshooting guide
├── networking/                  # Network troubleshooting guide
├── rbac/                        # RBAC troubleshooting guide
└── resources/                   # Resource management troubleshooting
```

## Quick Access to Guides

| Resource Type | Description | Link |
|---------------|-------------|------|
| **Pods** | Pod states, crashes, startup issues | [pods/README.md](./pods/README.md) |
| **Deployments** | Rolling updates, replica management | [deployments/README.md](./deployments/README.md) |
| **Services** | Connectivity, load balancing | [services/README.md](./services/README.md) |
| **Ingress** | Routing, SSL/TLS certificates | [ingress/README.md](./ingress/README.md) |
| **ConfigMaps & Secrets** | Configuration mounting, encoding | [configmaps-secrets/README.md](./configmaps-secrets/README.md) |
| **Nodes** | Node readiness, scheduling | [nodes/README.md](./nodes/README.md) |
| **Namespaces** | Namespace lifecycle, isolation | [namespaces/README.md](./namespaces/README.md) |
| **Storage** | PV, PVC, dynamic provisioning | [storage/README.md](./storage/README.md) |
| **Networking** | Pod communication, DNS | [networking/README.md](./networking/README.md) |
| **RBAC** | Permissions, service accounts | [rbac/README.md](./rbac/README.md) |
| **Resources** | Quotas, limits, resource management | [resources/README.md](./resources/README.md) |

## General Troubleshooting Commands

Essential kubectl commands for initial investigation across all resource types.

### Cluster Overview Commands

```bash
# Check cluster status
kubectl cluster-info
kubectl get nodes
kubectl get componentstatuses

# Check all resources in all namespaces
kubectl get all --all-namespaces

# Check events across the cluster
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Check resource usage
kubectl top nodes
kubectl top pods --all-namespaces
```

### Resource Inspection Commands

```bash
# Get detailed information about any resource
kubectl describe <resource-type> <resource-name> -n <namespace>

# Get resource configuration in YAML format
kubectl get <resource-type> <resource-name> -n <namespace> -o yaml

# Get resource configuration in JSON format
kubectl get <resource-type> <resource-name> -n <namespace> -o json

# Watch for real-time changes
kubectl get <resource-type> -n <namespace> -w

# Get logs from pods
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl logs <pod-name> -n <namespace> -f  # follow logs
```

### Debugging Commands

```bash
# Create a debug pod for troubleshooting
kubectl run debug --image=busybox --rm -it --restart=Never -- sh

# Execute commands in running pods
kubectl exec -it <pod-name> -n <namespace> -- <command>

# Port forward for testing connectivity
kubectl port-forward pod/<pod-name> <local-port>:<pod-port> -n <namespace>

# Copy files to/from pods
kubectl cp <local-path> <namespace>/<pod-name>:<pod-path>
kubectl cp <namespace>/<pod-name>:<pod-path> <local-path>
```

### Authentication and Authorization Commands

```bash
# Check what you can do
kubectl auth can-i <verb> <resource> -n <namespace>
kubectl auth can-i --list -n <namespace>

# Check as another user
kubectl auth can-i <verb> <resource> --as=<user> -n <namespace>
```

## Best Practices for Troubleshooting

### Systematic Approach

1. **Identify the scope**: Determine if the issue affects a single pod, service, or entire application
2. **Check recent changes**: Review recent deployments, configuration updates, or infrastructure changes
3. **Review logs and events**: Start with `kubectl describe` and `kubectl logs`
4. **Verify resource availability**: Check if cluster has sufficient CPU, memory, and storage
5. **Test connectivity and permissions**: Verify network connectivity and RBAC permissions

### Essential Tools

- **kubectl**: Primary Kubernetes CLI tool
- **stern**: Multi-pod log tailing (`stern <pod-pattern> -n <namespace>`)
- **k9s**: Interactive cluster management UI
- **kubectx/kubens**: Quick context and namespace switching
- **jq**: JSON processing for complex queries

### Monitoring and Observability

- Set up centralized logging (ELK stack, Fluentd, or similar)
- Use monitoring tools (Prometheus, Grafana)
- Implement distributed tracing for microservices
- Set up alerting for critical issues
- Regular health checks and synthetic monitoring

### Prevention Strategies

- Use resource quotas and limits to prevent resource exhaustion
- Implement proper health checks (liveness, readiness, startup probes)
- Use network policies for security
- Regular cluster maintenance and updates
- Backup and disaster recovery procedures
- Infrastructure as Code (IaC) for consistency

## Quick Reference Commands

```bash
# Force delete stuck resources (use with caution)
kubectl delete <resource> <name> -n <namespace> --grace-period=0 --force

# Get events sorted by time
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Get all resources of a specific type across all namespaces
kubectl get <resource-type> --all-namespaces

# Scale deployments
kubectl scale deployment <deployment-name> --replicas=<number> -n <namespace>

# Restart deployment (rolling restart)
kubectl rollout restart deployment/<deployment-name> -n <namespace>

# Check rollout status
kubectl rollout status deployment/<deployment-name> -n <namespace>

# Get resource usage
kubectl top nodes
kubectl top pods -n <namespace>

# Get specific field from resource
kubectl get pods -n <namespace> -o jsonpath='{.items[*].metadata.name}'
```

## Azure Kubernetes Service (AKS) Specific Troubleshooting

AKS provides additional Azure-specific features and integrations that require specialized troubleshooting approaches.

### AKS Cluster Management

```bash
# Get AKS cluster credentials
az aks get-credentials --resource-group <rg-name> --name <cluster-name>

# Check AKS cluster status
az aks show --resource-group <rg-name> --name <cluster-name> --query provisioningState

# Get AKS cluster upgrade versions
az aks get-upgrades --resource-group <rg-name> --name <cluster-name>

# Check AKS node pool status
az aks nodepool list --resource-group <rg-name> --cluster-name <cluster-name>

# Get AKS cluster configuration
az aks show --resource-group <rg-name> --name <cluster-name>
```

### AKS Node Troubleshooting

```bash
# Check node pool scaling
az aks nodepool scale --resource-group <rg-name> --cluster-name <cluster-name> --name <nodepool-name> --node-count <count>

# Check node pool upgrade status
az aks nodepool get-upgrades --resource-group <rg-name> --cluster-name <cluster-name> --nodepool-name <nodepool-name>

# Get node pool details
az aks nodepool show --resource-group <rg-name> --cluster-name <cluster-name> --name <nodepool-name>

# Check VM scale set issues (if using VMSS)
az vmss list-instances --resource-group <node-rg> --name <vmss-name>
az vmss get-instance-view --resource-group <node-rg> --name <vmss-name> --instance-id <id>
```

### AKS Networking Issues

```bash
# Check AKS network configuration
kubectl get configmap azure-ip-masq-agent-config -n kube-system -o yaml

# Verify Azure CNI configuration
kubectl describe configmap azure-cni-manager -n kube-system

# Check CoreDNS configuration in AKS
kubectl get configmap coredns -n kube-system -o yaml

# Test DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default.svc.cluster.local

# Check for network policies (if using Azure Network Policy)
kubectl get networkpolicies --all-namespaces
```

### AKS Storage Troubleshooting

```bash
# Check available storage classes
kubectl get storageclass

# Verify Azure disk/file provisioning
kubectl get pv -o custom-columns=NAME:.metadata.name,SIZE:.spec.capacity.storage,STORAGECLASS:.spec.storageClassName,STATUS:.status.phase

# Check CSI driver pods
kubectl get pods -n kube-system | grep csi

# Verify Azure Disk CSI driver
kubectl describe daemonset csi-azuredisk-node -n kube-system

# Verify Azure File CSI driver
kubectl describe daemonset csi-azurefile-node -n kube-system
```

### AKS Managed Identity Issues

```bash
# Check cluster managed identity
az aks show --resource-group <rg-name> --name <cluster-name> --query identity

# Verify pod identity (if using AAD Pod Identity)
kubectl get azureidentity --all-namespaces
kubectl get azureidentitybinding --all-namespaces

# Check workload identity (if using Workload Identity)
kubectl get serviceaccount <sa-name> -n <namespace> -o yaml
```

### AKS Add-ons Troubleshooting

```bash
# List enabled add-ons
az aks addon list --resource-group <rg-name> --name <cluster-name>

# Check monitoring add-on (Azure Monitor for containers)
kubectl get pods -n kube-system | grep omsagent

# Verify ingress controller (if using Application Gateway)
kubectl get pods -n kube-system | grep ingress

# Check virtual node add-on (if enabled)
kubectl get nodes | grep virtual-node

# Verify Key Vault CSI driver (if enabled)
kubectl get pods -n kube-system | grep csi-secrets-store
```

### Common AKS Issues and Solutions

#### Issue: Pods stuck in ContainerCreating with "failed to get sandbox image"
```bash
# Check node status and events
kubectl describe node <node-name>

# Verify container runtime
kubectl get nodes -o wide

# Check for image pull issues
kubectl describe pod <pod-name> -n <namespace>
```

#### Issue: Load balancer service not getting external IP
```bash
# Check service configuration
kubectl describe service <service-name> -n <namespace>

# Verify Azure load balancer
az network lb list --resource-group <node-rg>

# Check for subnet issues (in case of internal load balancer)
kubectl get service <service-name> -n <namespace> -o yaml
```

#### Issue: Cluster autoscaler not working
```bash
# Check cluster autoscaler logs
kubectl logs -f deployment/cluster-autoscaler -n kube-system

# Verify autoscaler configuration
kubectl describe configmap cluster-autoscaler-status -n kube-system

# Check node pool autoscaling settings
az aks nodepool show --resource-group <rg-name> --cluster-name <cluster-name> --name <nodepool-name> --query enableAutoScaling
```

#### Issue: Azure AD integration problems
```bash
# Check cluster Azure AD configuration
az aks show --resource-group <rg-name> --name <cluster-name> --query aadProfile

# Verify RBAC bindings
kubectl get clusterrolebindings | grep -i azure
kubectl get rolebindings --all-namespaces | grep -i azure

# Test authentication
kubectl auth can-i get pods --as=<azure-user>
```

### AKS Diagnostic Tools

```bash
# Enable AKS diagnostics
az aks enable-addons --resource-group <rg-name> --name <cluster-name> --addons monitoring

# Use kubectl-debug plugin for advanced debugging
kubectl debug <pod-name> -it --image=busybox

# Check AKS support bundle
# Use AKS Periscope for comprehensive cluster diagnostics
kubectl apply -f https://raw.githubusercontent.com/Azure/aks-periscope/main/deployment/aks-periscope.yaml
```

### Performance and Scaling Issues

```bash
# Check cluster metrics
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=memory
kubectl top pods --all-namespaces --sort-by=cpu

# Monitor resource quotas
kubectl describe resourcequota -n <namespace>

# Check horizontal pod autoscaler
kubectl get hpa --all-namespaces
kubectl describe hpa <hpa-name> -n <namespace>

# Verify vertical pod autoscaler (if enabled)
kubectl get vpa --all-namespaces
```

### AKS Security Troubleshooting

```bash
# Check Azure Policy compliance (if enabled)
kubectl get constrainttemplate
kubectl get constraints

# Verify Pod Security Policy (deprecated) or Pod Security Standards
kubectl get psp
kubectl get podsecuritypolicy

# Check network security
kubectl get networkpolicies --all-namespaces

# Verify image pull secrets
kubectl get secrets --field-selector type=kubernetes.io/dockerconfigjson --all-namespaces
```

### Useful AKS-specific kubectl plugins and tools

- **kubectl aks**: AKS-specific operations
- **kubectl cost**: Cost analysis for workloads
- **kubectl status**: Enhanced status checking
- **Azure Monitor for containers**: Container insights
- **AKS Periscope**: Comprehensive diagnostics

## Contributing

If you find issues not covered in this guide or have additional troubleshooting steps, please feel free to contribute by:

1. Creating an issue to discuss the problem
2. Submitting a pull request with additional troubleshooting steps
3. Sharing your troubleshooting experiences

## Additional Resources

- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Kubernetes Troubleshooting Guide](https://kubernetes.io/docs/tasks/debug-application-cluster/)

Remember: Always start with `kubectl describe` and `kubectl logs` - they provide the most valuable information for troubleshooting Kubernetes issues.
