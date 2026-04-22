# AGENTS.md — GitOps Repository for AKS Platform Addons

## Overview

ArgoCD App-of-Apps GitOps repository managing AKS cluster addons via Helm charts.
No application code — pure infrastructure-as-code (YAML/Helm templates only).
Deployed automatically by ArgoCD from `base_chart/`. Companion infra repo: `../03-plat-eng-aks-foundation/`.

## Repository Structure

```
base_chart/                  # Main Helm chart — creates ArgoCD Application CRs
  Chart.yaml                 # Chart metadata (control-plane-addons, v0.1.0)
  values.yaml                # Global config + per-addon enable/disable flags
  templates/
    {NN}-{addon-name}.yaml   # ArgoCD Application template per addon (NN = sync wave)

addon_charts/                # Individual addon Helm charts (19 addons)
  {addon-name}/
    Chart.yaml               # Chart metadata + upstream dependencies
    values.yaml              # Value overrides for upstream chart
    templates/               # (optional) Custom K8s resource templates
```

## Build / Lint / Validate Commands

```bash
make help             # Show available targets
make plugin-install   # Install helm-unittest Helm plugin
make deps             # Download Helm dependencies for charts with Chart.lock
make lint             # Lint all Helm charts (base_chart + addon_charts)
make template         # Render all Helm templates
make test             # Run helm-unittest tests
make snapshot-update  # Update helm-unittest snapshots
make all              # deps + lint + template + test
make package          # Package base_chart into .tgz
make clean            # Remove packaged .tgz files
```

## CI Pipeline

Workflow file: `.github/workflows/helm-ci.yml`

Triggers: push to `master`, pull_request to `master`

Steps:
1. Checkout repository
2. Install Helm
3. Install helm-unittest plugin
4. Download Helm dependencies for all addon charts (loop)
5. Lint `base_chart/`
6. Lint all addon charts (loop)
7. Render templates for `base_chart/`
8. Run helm-unittest tests for `base_chart/`
9. Run helm-unittest tests for all addon charts (loop)
10. Package `base_chart/` into `.tgz`
11. Login to GHCR
12. Push packaged chart to GHCR

## Architecture

ArgoCD watches `base_chart/` → each template creates an ArgoCD `Application` CR per addon →
each Application points to `addon_charts/<name>/` → ArgoCD deploys the addon's Helm chart.
Applications are gated by `{{- if (.Values.<addon_key>.enabled) }}`.

### Sync Wave Ordering (file prefix = sync wave = deploy order)

| Wave | Category | Examples |
|------|----------|---------|
| 0-4 | Infrastructure | resources, karpenter, metrics-server |
| 5-9 | Core services | kube-state-metrics, node-problem-detector, otel, cert-manager |
| 10-14 | Platform services | reloader, providers-config, cluster-secret, backup |
| 15-18 | Application services | kubecost, observability, cloudnative-pg, azure-service-operator |

## Code Style & Conventions

### Naming

| Context | Convention | Example |
|---------|-----------|---------|
| Values keys (base_chart) | `snake_case` | `cert_manager`, `kube_state_metrics` |
| Addon directory names | `kebab-case` | `cert-manager`, `kube-state-metrics` |
| Base chart template files | `{NN}-{kebab-case}.yaml` | `09-cert-manager.yaml`, `17-cloud-native-pg.yaml` |
| Addon template files | `{N}_{snake_case}.yaml` | `0_node_class.yaml`, `01_service.yaml` |
| Namespaces | `kebab-case` with `-system` suffix | `control-plane-system`, `resources-system` |

### Values Structure (`base_chart/values.yaml`)

```yaml
global:
  control_plane:
    namespace: control-plane-system
    project: addons-project
    repo: https://github.com/lurodrisilva/gitops
    deployment:
      limit: 5
      backoff: { duration: 240s, factor: 2, maxDuration: 10m }

<addon_key>:                 # snake_case
  addon_name: <addon-name>   # kebab-case, matches addon_charts/ directory
  enabled: true/false
  namespace: <target-namespace>
```

### Base Chart Template Pattern

Every `base_chart/templates/{NN}-{name}.yaml` follows this structure:

```yaml
---
{{- if (.Values.<addon_key>.enabled) }}
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: {{ .Values.<addon_key>.addon_name }}
  namespace: {{ .Values.global.control_plane.namespace }}
  annotations:
    argocd.argoproj.io/manifest-generate-paths: .
    argocd.argoproj.io/sync-wave: "<NN>"    # Must match file prefix number
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: {{ .Values.global.control_plane.project }}
  source:
    repoURL: {{ .Values.global.control_plane.repo }}
    targetRevision: HEAD
    path: {{ printf "addon_charts/%s" .Values.<addon_key>.addon_name }}
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: {{ .Values.<addon_key>.namespace }}
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

**Variations from the base pattern** (apply when needed):
- `ignoreDifferences` — for CRD-heavy addons (see cert-manager template)
- `ServerSideApply=true` — for addons with large CRDs (cloudnative-pg, azure-service-operator)

### Addon Chart Pattern (`addon_charts/<name>/`)

- **Chart.yaml**: `apiVersion: v2`, `type: application`. Pin dependency versions explicitly.
- **values.yaml**: Namespace upstream chart overrides under the dependency's name key.
- **templates/**: Only for custom resources not provided by upstream (e.g., Karpenter NodePool/NodeClass).

### YAML Formatting

- 2-space indentation throughout
- Document separator `---` at the top of template files
- Helm conditionals: `{{- if ... }}` with `{{- end }}` (dash-trimmed)
- Use `{{ printf "addon_charts/%s" .Values.<key>.addon_name }}` for path construction
- Comments for disabled/WIP items: inline `#` comments explaining why

### Git Commit Messages

Lowercase, present participle style:
- `adding <component> to the cluster`
- `fixing <component> deployment`
- `removing <component>`
- `fixing configurations`

## Adding a New Addon (Checklist)

1. Create `addon_charts/<addon-name>/Chart.yaml` and `values.yaml`
2. If upstream chart exists, add it as a `dependencies` entry in Chart.yaml
3. Run `helm dependency update addon_charts/<addon-name>/`
4. Create `base_chart/templates/{NN}-{addon-name}.yaml` using the template pattern above
5. Add entry to `base_chart/values.yaml` with `addon_name`, `enabled: false`, `namespace`
6. Validate: `helm lint base_chart/` and `helm template base_chart/`
7. Set `enabled: true` in values.yaml when ready to deploy

## Namespaces

| Namespace | Purpose |
|-----------|---------|
| `control-plane-system` | ArgoCD, most platform addons |
| `resources-system` | Crossplane resources, providers, cloudnative-pg, ASO |
| `karpenter` | Karpenter autoscaler |
| `backup-system` | Backup solutions |
| `messaging-system` | Strimzi Kafka operator |
| `testing-system` | k6 load-testing operator |
| `devops-system` | ArgoCD server (managed by infra repo) |

## Common Pitfalls

- **Sync wave mismatch**: File prefix `{NN}` MUST match the `argocd.argoproj.io/sync-wave` annotation value
- **addon_name vs key**: The values key is `snake_case` but `addon_name` is `kebab-case` matching the directory
- **Dependency not downloaded**: Run `helm dependency update` before linting addon charts with dependencies
- **CRD conflicts**: Use `ServerSideApply=true` and/or `ignoreDifferences` for CRD-heavy operators
- **Namespace in destination**: Some addons deploy to their own namespace (not `control-plane-system`); check the `namespace` field in values.yaml
