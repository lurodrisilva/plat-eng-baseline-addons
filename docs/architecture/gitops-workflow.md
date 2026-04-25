# GitOps Workflow

This document explains the GitOps workflow used in this repository.

## Overview

This repository implements the **App-of-Apps pattern** with ArgoCD:

1. Infrastructure deploys ArgoCD (via 01-aks-tf)
2. ArgoCD watches this repository's `base_chart/`
3. `base_chart` creates ArgoCD Applications for each enabled addon
4. Each Application deploys its addon from `addon_charts/`

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│ Git Repository (https://github.com/lurodrisilva/gitops)│
│                                                          │
│  base_chart/                                            │
│  ├── values.yaml (addon enabled flags)                 │
│  └── templates/ (ArgoCD Application CRDs)              │
│                                                          │
│  addon_charts/                                          │
│  ├── cert-manager/                                      │
│  ├── reloader/                                          │
│  └── ...                                                │
└─────────────────────────────────────────────────────────┘
                        ↓  (git pull)
┌─────────────────────────────────────────────────────────┐
│                      ArgoCD                              │
│                                                          │
│  Gitops Application (deployed by Terraform)             │
│  • Monitors: base_chart/                                │
│  • Auto-sync: enabled                                   │
│  • Prune: enabled                                       │
└─────────────────────────────────────────────────────────┘
                        ↓  (creates)
┌─────────────────────────────────────────────────────────┐
│           ArgoCD Applications (one per addon)            │
│                                                          │
│  For each addon where enabled: true in values.yaml     │
│  • cert-manager Application                             │
│  • reloader Application                                 │
│  • cloudnative-pg Application                           │
│  • azure-service-operator Application                   │
│  • strimzi-operator Application                         │
│  • k6-operator Application                              │
│  • wiremock Application                                 │
└─────────────────────────────────────────────────────────┘
                        ↓  (deploys)
┌─────────────────────────────────────────────────────────┐
│            Kubernetes Resources (addons)                 │
│                                                          │
│  • Deployments, Services, ConfigMaps, etc.             │
│  • Each addon in its designated namespace               │
└─────────────────────────────────────────────────────────┘
```

## Workflow Steps

### 1. Change Request

Developer modifies `base_chart/values.yaml`:

```yaml
metrics_server:
  enabled: true  # Changed from false
```

### 2. Commit and Push

```bash
git add base_chart/values.yaml
git commit -m "Enable metrics-server"
git push
```

### 3. ArgoCD Detection

ArgoCD polls the repository (~3 minutes):
- Detects change in base_chart/
- Compares desired state (Git) vs current state (cluster)
- Identifies drift

### 4. Automatic Sync

ArgoCD automatically syncs (due to `automated: true`):
- Renders base_chart Helm template
- Creates new Application CRD for metrics-server
- Applies to cluster

### 5. Addon Deployment

The new Application:
- Points to addon_charts/metrics-server/
- Resolves Helm dependencies
- Deploys resources to control-plane-system namespace

### 6. Continuous Monitoring

ArgoCD continuously:
- Monitors Git for changes
- Monitors cluster for drift
- Self-heals if resources are modified or deleted
- Prunes resources no longer in Git

## Key Concepts

### App-of-Apps Pattern

The `base_chart` is a "parent" application that creates "child" applications:

```
Gitops App (parent)
  ├── cert-manager App (child)
  ├── reloader App (child)
  ├── cloudnative-pg App (child)
  ├── azure-service-operator App (child)
  ├── strimzi-operator App (child)
  ├── k6-operator App (child)
  └── wiremock App (child)
```

Benefits:
- Single source of truth (values.yaml)
- Atomic addon management
- Organized structure
- Easy to add/remove addons

### Sync Policies

#### Automated Sync

```yaml
syncPolicy:
  automated:
    prune: true      # Delete resources not in Git
    selfHeal: true   # Revert manual changes
```

- Changes in Git automatically applied
- Manual cluster changes are reverted
- Deleted resources are pruned

#### Sync Waves

Templates use sync waves for ordering:

```yaml
annotations:
  argocd.argoproj.io/sync-wave: "9"
```

Lower numbers deploy first. Example order:
- Wave 0: Crossplane resources
- Wave 1: Karpenter
- Wave 2: Metrics server
- Wave 9: cert-manager
- Wave 10: reloader
- Wave 17: cloudnative-pg
- Wave 18: azure-service-operator
- Wave 19: strimzi-operator
- Wave 20: k6-operator
- Wave 21: wiremock

### Prune Behavior

When an addon is disabled:
1. Application CRD removed from cluster
2. ArgoCD deletes associated resources
3. Namespace may remain (depends on policy)

## GitOps Principles

This repository follows GitOps principles:

1. **Declarative**: Desired state in Git
2. **Versioned**: All changes in Git history
3. **Automated**: ArgoCD applies changes
4. **Auditable**: Git log shows all changes
5. **Self-healing**: Drift automatically corrected

## Comparison: Manual vs GitOps

### Manual Deployment

```bash
# Traditional approach
helm install metrics-server metrics-server/metrics-server \
  --namespace control-plane-system \
  --values custom-values.yaml
```

Problems:
- No audit trail
- Manual process
- Drift undetected
- No rollback
- Per-cluster configuration

### GitOps Deployment

```bash
# GitOps approach
git commit -m "Enable metrics-server"
git push
```

Benefits:
- Complete audit trail
- Automated deployment
- Drift detected and corrected
- Easy rollback (git revert)
- Same config for all clusters

## Rollback Procedure

To rollback a change:

```bash
# Find the commit to revert
git log --oneline

# Revert the change
git revert <commit-hash>

# Push
git push
```

ArgoCD will automatically sync the reverted state.

## Best Practices

1. **Small changes** - One addon per commit
2. **Meaningful commits** - Clear messages
3. **Test first** - Use dev environment
4. **Monitor sync** - Watch ArgoCD UI
5. **Use branches** - Feature branches for large changes
6. **Review PRs** - Peer review before merging
7. **Tag releases** - Version important states

## Troubleshooting

### Sync Not Happening

```bash
# Check ArgoCD is running
kubectl get pods -n devops-system

# Check application status
kubectl get application gitops -n control-plane-system

# Force refresh
kubectl patch application gitops -n control-plane-system \
  --type merge -p '{"operation":{"initiatedBy":{"username":"admin"}},"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

### Drift Detected But Not Healing

Check sync policy:

```bash
kubectl get application cert-manager -n control-plane-system \
  -o jsonpath='{.spec.syncPolicy.automated}' | jq
```

Should show `{"prune":true,"selfHeal":true}`.

## Related Documentation

- [Quickstart Guide](../setup/quickstart.md)
- [Enabling Addons](../guides/enabling-addons.md)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
