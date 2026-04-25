# Quickstart Guide

This guide will help you get started with the AKS Baseline GitOps repository.

## Prerequisites

- AKS cluster deployed via [01-aks-tf](../../../01-aks-tf/)
- ArgoCD installed and running
- kubectl configured to access the cluster
- Git access to this repository

## Quick Start

### 1. Verify Infrastructure

First, ensure the infrastructure is deployed:

```bash
# Check cluster access
kubectl get nodes

# Verify ArgoCD is running
kubectl get pods -n devops-system

# Check ArgoCD applications
kubectl get applications -n control-plane-system
```

### 2. Access ArgoCD

```bash
# Get ArgoCD URL
echo "http://luciano-argocd.eastus.cloudapp.azure.com"

# Get admin password
kubectl -n devops-system get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

### 3. Verify Base Chart Deployment

The base_chart should be automatically deployed as an ArgoCD Application:

```bash
# Check if gitops application exists
kubectl get application gitops -n control-plane-system

# View its status
kubectl describe application gitops -n control-plane-system
```

### 4. View Enabled Addons

Check which addons are currently deployed:

```bash
# List all addon applications
kubectl get applications -n control-plane-system

# Expected output (currently enabled):
# cert-manager
# reloader
# cloudnative-pg
# azure-service-operator
# strimzi-operator
# k6-operator
# wiremock
```

### 5. Enable Additional Addons

To enable more addons, edit `base_chart/values.yaml`:

```bash
# Clone the repository if not already done
git clone https://github.com/lurodrisilva/gitops.git
cd gitops

# Edit values.yaml
vim base_chart/values.yaml

# Change an addon from enabled: false to enabled: true
# For example:
metrics_server:
  addon_name: metrics-server
  enabled: true  # Change this
  namespace: control-plane-system
```

Commit and push:

```bash
git add base_chart/values.yaml
git commit -m "Enable metrics-server addon"
git push
```

ArgoCD will automatically detect the change and deploy the addon within ~3 minutes.

### 6. Manual Sync (Optional)

If you don't want to wait for automatic sync:

```bash
# Sync the base chart (gitops application)
kubectl patch application gitops -n control-plane-system \
  --type merge -p '{"operation":{"sync":{}}}'

# Or use ArgoCD CLI
argocd app sync gitops
```

## Understanding the Structure

### Base Chart (App-of-Apps)

The `base_chart` is a Helm chart that creates ArgoCD Applications:

```
base_chart/
├── Chart.yaml          # Helm chart metadata
├── values.yaml         # Addon configuration (enable/disable)
└── templates/          # ArgoCD Application CRDs
    ├── 00-resources.yaml
    ├── 09-cert-manager.yaml
    └── ...
```

### Addon Charts

Each addon has its own Helm chart in `addon_charts/`:

```
addon_charts/
└── cert-manager/
    ├── Chart.yaml      # Chart with upstream dependency
    └── values.yaml     # Custom configuration
```

## Common Tasks

### Check Addon Status

```bash
# List all addons
kubectl get applications -n control-plane-system

# Get detailed status
kubectl describe application cert-manager -n control-plane-system

# View sync status
kubectl get application cert-manager -n control-plane-system \
  -o jsonpath='{.status.sync.status}' && echo
```

### View Addon Resources

```bash
# View all resources for an addon
kubectl get all -n control-plane-system \
  -l app.kubernetes.io/instance=cert-manager

# View specific resource types
kubectl get deployments -n control-plane-system \
  -l app.kubernetes.io/instance=cert-manager
```

### Troubleshooting

#### Addon Not Deploying

```bash
# Check Application status
kubectl get application <addon-name> -n control-plane-system

# View Application events
kubectl describe application <addon-name> -n control-plane-system

# Check ArgoCD controller logs
kubectl logs -n devops-system \
  -l app.kubernetes.io/name=argocd-application-controller --tail=100
```

#### Sync Errors

```bash
# View sync status
kubectl get application <addon-name> -n control-plane-system \
  -o jsonpath='{.status.operationState}' | jq

# Force sync
kubectl patch application <addon-name> -n control-plane-system \
  --type merge -p '{"operation":{"sync":{"syncStrategy":{"hook":{"force":true}}}}}'
```

#### Addon Stuck

```bash
# Delete and let ArgoCD recreate
kubectl delete application <addon-name> -n control-plane-system

# Sync base chart to recreate
kubectl patch application gitops -n control-plane-system \
  --type merge -p '{"operation":{"sync":{}}}'
```

## Next Steps

- [Enable/Disable Addons](../guides/enabling-addons.md)
- [Add Custom Addons](../guides/adding-addons.md)
- [Understand GitOps Workflow](../architecture/gitops-workflow.md)
- [View Addon Reference](../reference/addon-list.md)

## ArgoCD UI

Access the ArgoCD UI at: http://luciano-argocd.eastus.cloudapp.azure.com

In the UI you can:
- View all applications and their sync status
- Manually trigger syncs
- View application logs and events
- Diff changes before syncing
- Rollback to previous versions

## Best Practices

1. **Always test locally** before pushing changes
   ```bash
   helm lint base_chart/
   helm template base_chart/ | kubectl apply --dry-run=client -f -
   ```

2. **Use meaningful commit messages**
   ```bash
   git commit -m "Enable metrics-server for resource monitoring"
   ```

3. **Monitor sync status** after changes
   ```bash
   watch kubectl get applications -n control-plane-system
   ```

4. **Review ArgoCD UI** for visual feedback on deployments

5. **Keep addons updated** by updating Chart.yaml versions in addon_charts/

## Support

For issues:
1. Check ArgoCD Application status
2. Review ArgoCD controller logs
3. Verify Helm chart syntax
4. Check addon-specific logs in their namespaces
