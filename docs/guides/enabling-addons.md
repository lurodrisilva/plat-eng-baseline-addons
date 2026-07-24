# Enabling and Disabling Addons

This guide explains how to enable or disable platform addons.

## Overview

Addons are controlled via `base_chart/values.yaml`. Each addon has an `enabled` flag that determines whether it's deployed.

## Enabling an Addon

### Step 1: Edit values.yaml

```bash
vim base_chart/values.yaml
```

Find the addon and change `enabled: false` to `enabled: true`:

```yaml
metrics_server:
  addon_name: metrics-server
  enabled: true              # Change from false to true
  namespace: control-plane-system
```

### Step 2: Commit and Push

```bash
git add base_chart/values.yaml
git commit -m "Enable metrics-server addon"
git push
```

### Step 3: Wait for Sync

ArgoCD automatically syncs within ~3 minutes. Monitor progress:

```bash
# Watch for new application
watch kubectl get applications -n control-plane-system

# Once it appears, check its status
kubectl describe application metrics-server -n control-plane-system
```

### Step 4: Verify Deployment

```bash
# Check if pods are running
kubectl get pods -n control-plane-system -l app.kubernetes.io/instance=metrics-server

# Verify the addon is functioning
kubectl top nodes  # For metrics-server specifically
```

## Disabling an Addon

### Step 1: Edit values.yaml

```bash
vim base_chart/values.yaml
```

Change `enabled: true` to `enabled: false`:

```yaml
metrics_server:
  addon_name: metrics-server
  enabled: false             # Change from true to false
  namespace: control-plane-system
```

### Step 2: Commit and Push

```bash
git add base_chart/values.yaml
git commit -m "Disable metrics-server addon"
git push
```

### Step 3: Verify Removal

ArgoCD will automatically prune (delete) the Application:

```bash
# Verify application is removed
kubectl get applications -n control-plane-system | grep metrics-server

# Verify resources are cleaned up
kubectl get all -n control-plane-system -l app.kubernetes.io/instance=metrics-server
```

## Manual Sync

To immediately sync changes instead of waiting:

```bash
# Sync the base chart application
kubectl patch application gitops -n control-plane-system \
  --type merge -p '{"operation":{"sync":{}}}'
```

## Currently Enabled Addons

Check `base_chart/values.yaml` for current status. As of now:

- cert-manager
- reloader
- cloudnative-pg
- azure-service-operator
- k6-operator
- wiremock

All others are disabled by default.

## Best Practices

1. **Enable one addon at a time** to easily troubleshoot issues
2. **Monitor resource usage** before enabling resource-intensive addons
3. **Review addon documentation** in `addon_charts/<name>/` before enabling
4. **Test in dev first** before enabling in production
5. **Keep values.yaml clean** - remove commented sections

## Troubleshooting

### Addon Not Appearing

```bash
# Check if gitops app synced successfully
kubectl get application gitops -n control-plane-system

# Force sync
kubectl patch application gitops -n control-plane-system \
  --type merge -p '{"operation":{"sync":{}}}'
```

### Addon Failed to Deploy

```bash
# Check application status
kubectl describe application <addon-name> -n control-plane-system

# View ArgoCD logs
kubectl logs -n devops-system -l app.kubernetes.io/name=argocd-application-controller
```

### Addon Not Pruning

```bash
# Manually delete the application
kubectl delete application <addon-name> -n control-plane-system

# Clean up remaining resources
kubectl delete all -n <namespace> -l app.kubernetes.io/instance=<addon-name>
```

## Related Documentation

- [Addon Reference](../reference/addon-list.md) - Complete addon list
- [Adding Addons](adding-addons.md) - Add custom addons
- [GitOps Workflow](../architecture/gitops-workflow.md) - Understanding the flow
