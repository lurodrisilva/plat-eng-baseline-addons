# Values Schema Reference

Complete reference for `base_chart/values.yaml` configuration.

## Global Configuration

```yaml
global:
  control_plane:
    namespace: control-plane-system      # Namespace for ArgoCD Applications
    project: addons-project              # ArgoCD project name
    repo: https://github.com/lurodrisilva/gitops  # This repository URL
    deployment:
      limit: 5                           # Maximum retry attempts
      backoff:
        duration: 240s                   # Initial backoff duration
        factor: 2                        # Backoff multiplier
        maxDuration: 10m                 # Maximum backoff duration
```

### Fields

- **namespace**: Namespace where ArgoCD Application CRDs are created
- **project**: ArgoCD project for RBAC and policies
- **repo**: Git repository URL for source
- **deployment.limit**: Number of sync retries before giving up
- **deployment.backoff**: Exponential backoff configuration for retries

## Addon Configuration Pattern

Each addon follows this pattern:

```yaml
addon_name:
  addon_name: <string>    # Required: Directory name in addon_charts/
  enabled: <boolean>      # Required: Enable/disable the addon
  namespace: <string>     # Required: Target namespace for deployment
```

### Fields

- **addon_name**: Must match the directory name in `addon_charts/`
- **enabled**: `true` to deploy, `false` to skip
- **namespace**: Kubernetes namespace where addon is deployed

## Complete Addon List

### Crossplane Resources

```yaml
resources:
  addon_name: resources
  enabled: false
  namespace: resources-system
```

Manages Crossplane-based Azure resources.

### Crossplane Providers

```yaml
providers:
  addon_name: providers
  enabled: false
  namespace: resources-system
```

Installs Crossplane providers (Azure, AWS, GCP).

### Provider Configuration

```yaml
providers_config:
  addon_name: providers-config
  enabled: false
  namespace: resources-system
```

Configures authentication for Crossplane providers.

### Backup Solutions

```yaml
backup:
  addon_name: backup
  enabled: false
  namespace: backup-system
```

Backup and disaster recovery (not yet configured).

### Sample Addon

```yaml
sample:
  addon_name: sample
  enabled: false
  namespace: backup-system
```

Example addon for testing purposes.

### Karpenter

```yaml
karpenter:
  addon_name: karpenter
  enabled: false
  namespace: karpenter
```

Kubernetes cluster autoscaler (currently deployed via Terraform).

### Metrics Server

```yaml
metrics_server:
  addon_name: metrics-server
  enabled: false
  namespace: control-plane-system
```

Core resource metrics API for HPA and kubectl top.

### Kube State Metrics

```yaml
kube_state_metrics:
  addon_name: kube-state-metrics
  enabled: false
  namespace: control-plane-system
```

Kubernetes object state metrics for monitoring.

### Node Problem Detector

```yaml
node_problem_detector:
  addon_name: node-problem-detector
  enabled: false
  namespace: control-plane-system
```

Detects and reports node-level issues.

### Cert Manager

```yaml
cert_manager:
  addon_name: cert-manager
  enabled: true
  namespace: control-plane-system
```

TLS certificate management and automation.

### OpenTelemetry Operator

```yaml
opentelemetry_operator:
  addon_name: opentelemetry-operator
  enabled: false
  namespace: control-plane-system
```

Manages OpenTelemetry collectors and instrumentation.

### OpenTelemetry Collector

```yaml
opentelemetry_collector:
  addon_name: opentelemetry-collector
  enabled: false
  namespace: control-plane-system
```

Collects and exports telemetry data.

### Datadog Operator

```yaml
datadog_operator:
  addon_name: datadog-operator
  enabled: false
  namespace: control-plane-system
```

Datadog monitoring integration.

### Reloader

```yaml
reloader:
  addon_name: reloader
  enabled: true
  namespace: control-plane-system
```

Auto-restarts pods when ConfigMaps/Secrets change.

### Cluster Secret

```yaml
cluster_secret:
  addon_name: cluster-secret
  enabled: false
  namespace: control-plane-system
```

Replicates secrets across namespaces.

### Kubecost

```yaml
kube_cost:
  addon_name: kubecost
  enabled: false
  namespace: control-plane-system
```

Kubernetes cost monitoring and optimization.

### Observability Stack

```yaml
observability:
  addon_name: observability
  enabled: false
  namespace: control-plane-system
```

Complete monitoring stack (Prometheus, Grafana, Loki).

### CloudNative PostgreSQL

```yaml
cloudnative_pg:
  addon_name: cloudnative-pg
  enabled: true
  namespace: resources-system
```

PostgreSQL operator for Kubernetes.

### Azure Service Operator

```yaml
azure_service_operator:
  addon_name: azure-service-operator
  enabled: true
  namespace: resources-system
```

Manages Azure resources directly from Kubernetes via CRDs. Supports workload identity authentication, service principals, and managed identities.

### Strimzi Kafka Operator

```yaml
strimzi_operator:
  addon_name: strimzi-operator
  enabled: true
  namespace: messaging-system
```

Strimzi operator for managing Kafka clusters, topics, and users on Kubernetes. Watches all namespaces. Uses `ServerSideApply=true` for its CRD bundle.

### k6 Operator

```yaml
k6_operator:
  addon_name: k6-operator
  enabled: true
  namespace: testing-system
```

Grafana k6 operator for distributed load testing via `TestRun` CRDs. Uses `ServerSideApply=true` for its CRD bundle.

### WireMock

```yaml
wiremock:
  addon_name: wiremock
  enabled: true
  namespace: testing-system
```

Shared WireMock mock server for test/preview tiers. Single multi-tenant Deployment in `testing-system`; consumer apps register stubs via the WireMock Admin API at install/upgrade. NetworkPolicy gates access to namespaces labelled `wiremock.io/consumer=true`. **Do not enable on production clusters** — `/__admin/*` has no built-in auth.

## Example Configuration

### Minimal (Default)

```yaml
global:
  control_plane:
    namespace: control-plane-system
    project: addons-project
    repo: https://github.com/lurodrisilva/gitops
    deployment:
      limit: 5
      backoff:
        duration: 240s
        factor: 2
        maxDuration: 10m

cert_manager:
  addon_name: cert-manager
  enabled: true
  namespace: control-plane-system

reloader:
  addon_name: reloader
  enabled: true
  namespace: control-plane-system

cloudnative_pg:
  addon_name: cloudnative-pg
  enabled: true
  namespace: resources-system

azure_service_operator:
  addon_name: azure-service-operator
  enabled: true
  namespace: resources-system

strimzi_operator:
  addon_name: strimzi-operator
  enabled: true
  namespace: messaging-system

k6_operator:
  addon_name: k6-operator
  enabled: true
  namespace: testing-system

wiremock:
  addon_name: wiremock
  enabled: true
  namespace: testing-system
```

### Full Observability

```yaml
metrics_server:
  addon_name: metrics-server
  enabled: true
  namespace: control-plane-system

kube_state_metrics:
  addon_name: kube-state-metrics
  enabled: true
  namespace: control-plane-system

observability:
  addon_name: observability
  enabled: true
  namespace: control-plane-system
```

## Validation

### Required Fields

Every addon must have:
- `addon_name` (string, matches directory)
- `enabled` (boolean)
- `namespace` (string, valid Kubernetes namespace)

### Naming Conventions

- Addon names use `snake_case` in values.yaml
- Directory names use `kebab-case` in addon_charts/
- Example: `cert_manager` in values → `cert-manager` directory

## Best Practices

1. **Start minimal** - Only enable what you need
2. **One at a time** - Enable addons incrementally
3. **Monitor resources** - Check cluster capacity before enabling
4. **Use comments** - Document why addons are enabled/disabled
5. **Version control** - Track all changes in Git

## Troubleshooting

### Addon Not Deploying

Check values.yaml syntax:

```bash
# Validate YAML
yamllint base_chart/values.yaml

# Test Helm template rendering
helm template base_chart/
```

### Name Mismatch

Ensure `addon_name` matches directory:

```bash
# List addon directories
ls addon_charts/

# Check values.yaml
grep "addon_name:" base_chart/values.yaml
```

## Related Documentation

- [Addon Reference](addon-list.md)
- [Enabling Addons](../guides/enabling-addons.md)
- [Adding Addons](../guides/adding-addons.md)
