# Adding New Addons

This guide shows you how to add custom addons to the platform.

## Prerequisites

- Understanding of Helm charts
- Access to push to the repository
- Addon Helm chart or ability to create one

## Steps to Add a New Addon

### 1. Create Addon Chart Directory

```bash
mkdir -p addon_charts/my-addon
cd addon_charts/my-addon
```

### 2. Create Chart.yaml

```yaml
apiVersion: v2
name: my-addon
description: My custom addon for the platform
type: application
version: 0.1.0
appVersion: "1.0.0"

# Add upstream chart as dependency
dependencies:
  - name: upstream-chart-name
    version: 1.2.3
    repository: https://charts.example.com
```

### 3. Create values.yaml

Add custom configuration overrides:

```yaml
# Override upstream chart values
upstream-chart-name:
  replicaCount: 2
  resources:
    limits:
      cpu: 200m
      memory: 256Mi
    requests:
      cpu: 100m
      memory: 128Mi
```

### 4. Create ArgoCD Application Template

Create `base_chart/templates/XX-my-addon.yaml` (where XX is sync wave number):

```yaml
---
{{- if (.Values.my_addon.enabled) }}
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: {{ .Values.my_addon.addon_name }}
  namespace: {{ .Values.global.control_plane.namespace }}
  annotations:
    argocd.argoproj.io/manifest-generate-paths: .
    argocd.argoproj.io/sync-wave: "15"  # Must match file prefix number
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: {{ .Values.global.control_plane.project }}
  source:
    repoURL: {{ .Values.global.control_plane.repo }}
    targetRevision: HEAD
    path: {{ printf "addon_charts/%s" .Values.my_addon.addon_name }}
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: {{ .Values.my_addon.namespace }}
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - Validate=false
      - Prune=true
      - ApplyOutOfSyncOnly=true
      - Force=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    {{- with .Values.global.control_plane.deployment }}
    retry:
      limit: {{ .limit }}
      backoff:
        duration: {{ .backoff.duration }}
        factor: {{ .backoff.factor }}
        maxDuration: {{ .backoff.maxDuration }}
    {{- end }}
{{- end }}
```

**Optional variations** (apply when needed):
- Add `ignoreDifferences` for CRD-heavy addons (see `09-cert-manager.yaml`)
- Add `ServerSideApply=true` to syncOptions for large CRDs (see `17-cloud-native-pg.yaml`, `18-azure-service-operator.yaml`, `19-strimzi-operator.yaml`, `20-k6-operator.yaml`)

### 5. Add Configuration to base_chart/values.yaml

```yaml
my_addon:
  addon_name: my-addon
  enabled: false              # Start disabled
  namespace: my-addon-system
```

### 6. Test Locally

```bash
# Lint the base chart
helm lint base_chart/

# Template and check output
helm template base_chart/ --values base_chart/values.yaml | grep -A 50 my-addon

# Update dependencies
helm dependency update addon_charts/my-addon/
```

### 7. Commit and Push

```bash
git add addon_charts/my-addon/ base_chart/
git commit -m "adding my-addon to the cluster"
git push
```

### 8. Enable and Test

Edit `base_chart/values.yaml` to enable:

```yaml
my_addon:
  enabled: true
```

Commit, push, and monitor deployment.

## Sync Wave Numbers

Organize addons with sync waves for proper ordering:

- 0-4: Infrastructure (resources, karpenter, metrics-server, providers, kube-state-metrics)
- 5-9: Core services (node-problem-detector, otel, datadog, cert-manager)
- 10-14: Platform services (reloader, providers-config, backup, cluster-secret)
- 15-18: Application services (kubecost, observability, cloudnative-pg, azure-service-operator)
- 19+: Workload operators (strimzi-operator, k6-operator)

## Best Practices

1. **Use dependencies** - Reference upstream charts when possible
2. **Minimal values** - Only override what's necessary
3. **Resource limits** - Always set requests and limits
4. **Namespaces** - Use dedicated namespaces for isolation
5. **Testing** - Test locally before committing
6. **Documentation** - Add README in addon_charts/my-addon/
7. **Versioning** - Pin dependency versions in Chart.yaml

## Example: Adding Prometheus

See `addon_charts/observability/` for a complete example of a complex addon.

## Troubleshooting

### Dependency Download Fails

```bash
# Update dependencies manually
helm dependency update addon_charts/my-addon/

# Check Chart.lock file is created
ls addon_charts/my-addon/Chart.lock
```

### Template Rendering Issues

```bash
# Test template rendering
helm template my-addon addon_charts/my-addon/

# Check for YAML syntax
yamllint addon_charts/my-addon/values.yaml
```

## Related Documentation

- [Enabling Addons](enabling-addons.md)
- [Values Schema](../reference/values-schema.md)
- [Addon Reference](../reference/addon-list.md)
