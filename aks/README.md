# Azure Kubernetes Service (AKS) Troubleshooting Guide

A comprehensive guide for troubleshooting Azure Kubernetes Service (AKS) specific issues, including Azure integrations like App Configuration and Key Vault.

## Table of Contents

- [AKS Cluster Management](#aks-cluster-management)
- [AKS Node Troubleshooting](#aks-node-troubleshooting)
- [AKS Networking Issues](#aks-networking-issues)
- [AKS Storage Troubleshooting](#aks-storage-troubleshooting)
- [AKS Managed Identity Issues](#aks-managed-identity-issues)
- [AKS Add-ons Troubleshooting](#aks-add-ons-troubleshooting)
- [Azure App Configuration Integration](#azure-app-configuration-integration)
- [Azure Key Vault Integration](#azure-key-vault-integration)
- [Common AKS Issues and Solutions](#common-aks-issues-and-solutions)
- [AKS Diagnostic Tools](#aks-diagnostic-tools)
- [Performance and Scaling Issues](#performance-and-scaling-issues)
- [AKS Security Troubleshooting](#aks-security-troubleshooting)

## AKS Cluster Management

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

## AKS Node Troubleshooting

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

## AKS Networking Issues

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

## AKS Storage Troubleshooting

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

## AKS Managed Identity Issues

```bash
# Check cluster managed identity
az aks show --resource-group <rg-name> --name <cluster-name> --query identity

# Verify pod identity (if using AAD Pod Identity)
kubectl get azureidentity --all-namespaces
kubectl get azureidentitybinding --all-namespaces

# Check workload identity (if using Workload Identity)
kubectl get serviceaccount <sa-name> -n <namespace> -o yaml

# Verify managed identity assignments
az role assignment list --assignee <identity-client-id> --output table
```

## AKS Add-ons Troubleshooting

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

## Azure App Configuration Integration

Azure App Configuration provides centralized configuration management for AKS applications.

### Prerequisites and Setup

```bash
# Install App Configuration Kubernetes Provider
helm repo add azureappconfig https://azureappconfig.github.io/kubernetes-provider/
helm repo update
helm install azureappconfig azureappconfig/azureappconfig-kubernetes-provider

# Check provider installation
kubectl get pods -n azureappconfig-system
kubectl logs -f deployment/azureappconfig-kubernetes-provider -n azureappconfig-system
```

### Common App Configuration Issues

#### Issue: ConfigMap not syncing from App Configuration

```bash
# Check AzureAppConfigurationProvider resource
kubectl get acp -n <namespace>
kubectl describe acp <provider-name> -n <namespace>

# Verify authentication (using managed identity)
kubectl get serviceaccount <sa-name> -n <namespace> -o yaml

# Check provider logs
kubectl logs -f deployment/azureappconfig-kubernetes-provider -n azureappconfig-system

# Verify App Configuration connection
az appconfig show --name <app-config-name> --resource-group <rg-name>
```

#### Issue: Authentication failures with App Configuration

```bash
# Check managed identity permissions
az role assignment list --assignee <identity-client-id> --scope /subscriptions/<sub-id>/resourceGroups/<rg-name>/providers/Microsoft.AppConfiguration/configurationStores/<store-name>

# Required roles for App Configuration
# - App Configuration Data Reader
az role assignment create \
  --assignee <identity-client-id> \
  --role "App Configuration Data Reader" \
  --scope /subscriptions/<sub-id>/resourceGroups/<rg-name>/providers/Microsoft.AppConfiguration/configurationStores/<store-name>

# Check workload identity configuration
kubectl describe serviceaccount <sa-name> -n <namespace>
kubectl get federatedidentity -n <namespace>
```

#### Issue: Key-Value pairs not reflecting in ConfigMap

```bash
# Check AzureAppConfigurationProvider configuration
kubectl get acp <provider-name> -n <namespace> -o yaml

# Verify key filters and label filters
# Example ACP resource
cat <<EOF | kubectl apply -f -
apiVersion: azconfig.io/v1
kind: AzureAppConfigurationProvider
metadata:
  name: app-config-provider
  namespace: <namespace>
spec:
  endpoint: https://<app-config-name>.azconfig.io
  target:
    configMapName: app-config-configmap
  auth:
    workloadIdentity:
      serviceAccountName: <sa-name>
  configuration:
    selectors:
      - keyFilter: "myapp:*"
        labelFilter: "production"
EOF

# Check if ConfigMap is created and updated
kubectl get configmap app-config-configmap -n <namespace> -o yaml
kubectl describe configmap app-config-configmap -n <namespace>
```

#### Issue: App Configuration refresh not working

```bash
# Check refresh settings in ACP
kubectl get acp <provider-name> -n <namespace> -o yaml | grep -A 10 refresh

# Verify sentinel key configuration
az appconfig kv set --name <app-config-name> --key "sentinel" --value "1" --label "production"

# Monitor refresh events
kubectl get events -n <namespace> --field-selector involvedObject.name=<provider-name>
```

### App Configuration Monitoring and Debugging

```bash
# Enable debug logging for App Configuration provider
kubectl patch deployment azureappconfig-kubernetes-provider -n azureappconfig-system -p '{"spec":{"template":{"spec":{"containers":[{"name":"manager","args":["--zap-log-level=debug"]}]}}}}'

# Check App Configuration metrics (if monitoring is enabled)
kubectl get servicemonitor -n azureappconfig-system

# Verify network connectivity to App Configuration
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup <app-config-name>.azconfig.io
```

## Azure Key Vault Integration

Azure Key Vault integration with AKS can be achieved through the Key Vault CSI driver or Key Vault FlexVolume (deprecated).

### Key Vault CSI Driver Setup and Troubleshooting

```bash
# Check if Key Vault CSI driver is installed
kubectl get pods -n kube-system -l app=secrets-store-csi-driver
kubectl get pods -n kube-system -l app=secrets-store-provider-azure

# Install Key Vault CSI driver (if not installed)
helm repo add csi-secrets-store-provider-azure https://azure.github.io/secrets-store-csi-driver-provider-azure/charts
helm install csi-secrets-store-provider-azure csi-secrets-store-provider-azure/csi-secrets-store-provider-azure

# Check CSI driver logs
kubectl logs -f daemonset/secrets-store-csi-driver -n kube-system
kubectl logs -f daemonset/secrets-store-provider-azure -n kube-system
```

### Common Key Vault Issues

#### Issue: Secrets not mounting from Key Vault

```bash
# Check SecretProviderClass configuration
kubectl get secretproviderclass -n <namespace>
kubectl describe secretproviderclass <spc-name> -n <namespace>

# Example SecretProviderClass
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault
  namespace: <namespace>
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    userAssignedIdentityID: "<identity-client-id>"
    keyvaultName: "<keyvault-name>"
    objects: |
      array:
        - |
          objectName: secret1
          objectType: secret
        - |
          objectName: key1
          objectType: key
        - |
          objectName: cert1
          objectType: cert
    tenantId: "<tenant-id>"
EOF

# Check pod mounting the secrets
kubectl describe pod <pod-name> -n <namespace>
kubectl get events -n <namespace> --field-selector involvedObject.name=<pod-name>
```

#### Issue: Authentication failures with Key Vault

```bash
# Check managed identity permissions for Key Vault
az keyvault show --name <keyvault-name> --resource-group <rg-name>

# Verify Key Vault access policies or RBAC permissions
az keyvault list-permissions --name <keyvault-name>

# For RBAC-enabled Key Vault, check role assignments
az role assignment list --assignee <identity-client-id> --scope /subscriptions/<sub-id>/resourceGroups/<rg-name>/providers/Microsoft.KeyVault/vaults/<keyvault-name>

# Required roles for Key Vault
# - Key Vault Secrets User (for secrets)
# - Key Vault Certificate User (for certificates)  
# - Key Vault Crypto User (for keys)
az role assignment create \
  --assignee <identity-client-id> \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/<sub-id>/resourceGroups/<rg-name>/providers/Microsoft.KeyVault/vaults/<keyvault-name>
```

#### Issue: Workload Identity not working with Key Vault

```bash
# Check workload identity configuration
kubectl get serviceaccount <sa-name> -n <namespace> -o yaml

# Verify service account annotations
kubectl annotate serviceaccount <sa-name> azure.workload.identity/client-id=<identity-client-id> -n <namespace>

# Check federated identity credential
az identity federated-credential list --name <identity-name> --resource-group <rg-name>

# Create federated identity credential (if missing)
az identity federated-credential create \
  --name <credential-name> \
  --identity-name <identity-name> \
  --resource-group <rg-name> \
  --issuer <oidc-issuer-url> \
  --subject system:serviceaccount:<namespace>:<sa-name>

# Get OIDC issuer URL
az aks show --resource-group <rg-name> --name <cluster-name> --query "oidcIssuerProfile.issuerUrl" -o tsv
```

#### Issue: Certificate rotation not working

```bash
# Check if auto-rotation is enabled
kubectl get secretproviderclass <spc-name> -n <namespace> -o yaml | grep rotation

# Enable rotation in SecretProviderClass
kubectl patch secretproviderclass <spc-name> -n <namespace> --type='merge' -p='{"spec":{"secretObjects":[{"secretName":"<secret-name>","type":"Opaque","data":[{"objectName":"<object-name>","key":"<key-name>"}]}]}}'

# Check rotation events
kubectl get events -n <namespace> | grep rotation

# Verify certificate expiry
kubectl get secret <secret-name> -n <namespace> -o json | jq -r '.data."tls.crt"' | base64 -d | openssl x509 -dates -noout
```

### Key Vault Monitoring and Debugging

```bash
# Enable debug logging for CSI driver
kubectl patch daemonset secrets-store-csi-driver -n kube-system -p='{"spec":{"template":{"spec":{"containers":[{"name":"secrets-store","args":["--endpoint=$(CSI_ENDPOINT)","--nodeid=$(KUBE_NODE_NAME)","--provider-volume=/etc/kubernetes/secrets-store-csi-providers","--additional-metadata-labels=secrets-store.csi.k8s.io/pod.name,secrets-store.csi.k8s.io/pod.namespace,secrets-store.csi.k8s.io/pod.uid,secrets-store.csi.k8s.io/pod.service-account.name","--provider-health-check=true","--provider-health-check-interval=2m","--v=5"]}]}}}}'

# Check Key Vault network access
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup <keyvault-name>.vault.azure.net

# Test Key Vault access from pod
kubectl run -it --rm debug --image=mcr.microsoft.com/azure-cli --restart=Never -- sh
# Inside the pod:
# az login --identity --username <identity-client-id>
# az keyvault secret show --vault-name <keyvault-name> --name <secret-name>
```

## Common AKS Issues and Solutions

### Issue: Pods stuck in ContainerCreating with "failed to get sandbox image"

```bash
# Check node status and events
kubectl describe node <node-name>

# Verify container runtime
kubectl get nodes -o wide

# Check for image pull issues
kubectl describe pod <pod-name> -n <namespace>
```

### Issue: Load balancer service not getting external IP

```bash
# Check service configuration
kubectl describe service <service-name> -n <namespace>

# Verify Azure load balancer
az network lb list --resource-group <node-rg>

# Check for subnet issues (in case of internal load balancer)
kubectl get service <service-name> -n <namespace> -o yaml
```

### Issue: Cluster autoscaler not working

```bash
# Check cluster autoscaler logs
kubectl logs -f deployment/cluster-autoscaler -n kube-system

# Verify autoscaler configuration
kubectl describe configmap cluster-autoscaler-status -n kube-system

# Check node pool autoscaling settings
az aks nodepool show --resource-group <rg-name> --cluster-name <cluster-name> --name <nodepool-name> --query enableAutoScaling
```

### Issue: Azure AD integration problems

```bash
# Check cluster Azure AD configuration
az aks show --resource-group <rg-name> --name <cluster-name> --query aadProfile

# Verify RBAC bindings
kubectl get clusterrolebindings | grep -i azure
kubectl get rolebindings --all-namespaces | grep -i azure

# Test authentication
kubectl auth can-i get pods --as=<azure-user>
```

## AKS Diagnostic Tools

```bash
# Enable AKS diagnostics
az aks enable-addons --resource-group <rg-name> --name <cluster-name> --addons monitoring

# Use kubectl-debug plugin for advanced debugging
kubectl debug <pod-name> -it --image=busybox

# Check AKS support bundle
# Use AKS Periscope for comprehensive cluster diagnostics
kubectl apply -f https://raw.githubusercontent.com/Azure/aks-periscope/main/deployment/aks-periscope.yaml
```

## Performance and Scaling Issues

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

## AKS Security Troubleshooting

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

## Useful AKS-specific Tools and Commands

### Essential Tools

- **kubectl aks**: AKS-specific operations
- **kubectl cost**: Cost analysis for workloads
- **kubectl status**: Enhanced status checking
- **Azure Monitor for containers**: Container insights
- **AKS Periscope**: Comprehensive diagnostics

### Monitoring Commands

```bash
# Check cluster cost analysis
kubectl cost --window 7d --aggregate namespace

# View container insights
az monitor log-analytics query \
  --workspace <workspace-id> \
  --analytics-query "ContainerInventory | where TimeGenerated > ago(1h) | summarize count() by Computer, Image"

# Check node resource usage over time
az monitor metrics list \
  --resource /subscriptions/<sub-id>/resourceGroups/<node-rg>/providers/Microsoft.Compute/virtualMachineScaleSets/<vmss-name> \
  --metric "Percentage CPU"
```

## Best Practices for AKS Troubleshooting

1. **Use Azure Monitor for Containers**: Enable container insights for comprehensive monitoring
2. **Implement proper RBAC**: Use Azure AD integration with Kubernetes RBAC
3. **Use Workload Identity**: Prefer Workload Identity over Pod Identity for better security
4. **Monitor costs**: Use cost management tools to track resource usage
5. **Keep clusters updated**: Regularly update AKS clusters and node pools
6. **Use diagnostic tools**: Leverage AKS Periscope and other diagnostic tools
7. **Network policies**: Implement network policies for microsegmentation
8. **Resource management**: Set appropriate resource requests and limits

## Additional Resources

- [AKS Troubleshooting Documentation](https://docs.microsoft.com/en-us/azure/aks/troubleshooting)
- [Azure Key Vault CSI Driver](https://azure.github.io/secrets-store-csi-driver-provider-azure/)
- [Azure App Configuration Kubernetes Provider](https://docs.microsoft.com/en-us/azure/azure-app-configuration/quickstart-azure-kubernetes-service)
- [AKS Workload Identity](https://docs.microsoft.com/en-us/azure/aks/workload-identity-overview)
- [AKS Best Practices](https://docs.microsoft.com/en-us/azure/aks/best-practices)