# Addon Reference

Complete reference for all available addons in the platform.

## Currently Enabled Addons

### cert-manager

**Purpose**: Automates TLS certificate management
**Namespace**: control-plane-system
**Version**: v1.16.2
**Upstream**: https://charts.jetstack.io

Features:
- Automatic certificate issuance and renewal
- Support for Let's Encrypt, Vault, and other issuers
- CRD-based certificate management

Configuration:
```yaml
cert-manager:
  replicaCount: 2
  crds:
    enabled: true
```

### reloader

**Purpose**: Auto-restarts pods when ConfigMaps/Secrets change
**Namespace**: control-plane-system
**Upstream**: https://stakater.github.io/Reloader/

Features:
- Watches ConfigMaps and Secrets
- Triggers rolling restarts automatically
- Annotation-based configuration

### cloudnative-pg

**Purpose**: PostgreSQL operator for Kubernetes
**Namespace**: resources-system
**Upstream**: https://cloudnative-pg.io

Features:
- High-availability PostgreSQL clusters
- Automated backups and recovery
- Connection pooling with PgBouncer

### azure-service-operator

**Purpose**: Manage Azure resources directly from Kubernetes
**Namespace**: resources-system
**Version**: v2.18.0
**Upstream**: https://github.com/Azure/azure-service-operator

Features:
- Declarative Azure resource management via Kubernetes CRDs
- Workload identity and service principal authentication
- Supports Azure databases, networking, storage, and more

### strimzi-operator

**Purpose**: Strimzi Kafka operator — deploys and manages Kafka clusters, topics, and users on Kubernetes
**Namespace**: messaging-system
**Version**: 0.51.0
**Upstream**: https://strimzi.io/charts/

Features:
- Declarative Kafka cluster management via `Kafka` / `KafkaTopic` / `KafkaUser` CRDs
- Watches all namespaces (`watchAnyNamespace: true`)
- Leader election for safe multi-replica operation
- `ServerSideApply=true` for large CRDs

### k6-operator

**Purpose**: Grafana k6 Kubernetes operator — runs distributed load tests against cluster workloads
**Namespace**: testing-system
**Version**: chart 4.3.2 / appVersion 1.3.2
**Upstream**: https://grafana.github.io/helm-charts

Features:
- Declarative load tests via `TestRun` CRDs
- Optional `PrivateLoadZone` CRD for Grafana Cloud integration
- Cluster-wide RBAC — reconciles `TestRun` resources in any namespace
- `ServerSideApply=true` required for the CRD bundle

### wiremock

**Purpose**: Shared HTTP API mock server for test/preview tiers — every consumer app points at this single instance instead of real third-party HTTP dependencies
**Namespace**: testing-system
**Version**: chart 0.1.0 / appVersion 3.9.1
**Upstream**: https://hub.docker.com/r/wiremock/wiremock (custom chart, no upstream Helm chart)

Features:
- Single multi-tenant Deployment, `strategy.type: Recreate` (in-memory mappings are not replicated)
- ClusterIP Service on `wiremock.testing-system.svc.cluster.local:8080`
- NetworkPolicy default-deny — only namespaces labelled `wiremock.io/consumer=true` may reach the Admin API
- Hardened pod: non-root, `readOnlyRootFilesystem`, drops all capabilities, no PVC
- Bounded memory: `--no-request-journal`, `--max-request-journal-entries=1000`
- Stubs are owned by consuming applications and registered at install/upgrade via `POST /__admin/mappings`; nothing is baked into this chart

> **Production warning**: WireMock has no built-in auth on `/__admin/*`. The NetworkPolicy is the only enforcement boundary. Do not enable on a production cluster.

## Available Addons (Disabled)

### resources

**Purpose**: Crossplane managed Azure resources
**Namespace**: resources-system
**Status**: Disabled

Deploys Crossplane custom resources for Azure infrastructure.

### providers

**Purpose**: Crossplane provider installations
**Namespace**: resources-system
**Status**: Disabled

Installs Crossplane providers (Azure, AWS, GCP).

### providers-config

**Purpose**: Crossplane provider configurations
**Namespace**: resources-system
**Status**: Disabled

Configures authentication and settings for Crossplane providers.

### karpenter

**Purpose**: Kubernetes cluster autoscaler
**Namespace**: karpenter
**Status**: Disabled
**Note**: Currently deployed via Terraform

Advanced node autoscaling with custom scheduling.

### metrics-server

**Purpose**: Core resource metrics API
**Namespace**: control-plane-system
**Status**: Disabled

Provides CPU/memory metrics for horizontal pod autoscaling.

Enable for:
- `kubectl top nodes/pods`
- Horizontal Pod Autoscaler (HPA)
- Vertical Pod Autoscaler (VPA)

### kube-state-metrics

**Purpose**: Kubernetes object state metrics
**Namespace**: control-plane-system
**Status**: Disabled

Exposes metrics about Kubernetes objects (deployments, pods, nodes).

### node-problem-detector

**Purpose**: Node health monitoring
**Namespace**: control-plane-system
**Status**: Disabled

Detects and reports node issues (disk pressure, network issues, etc.).

### opentelemetry-operator

**Purpose**: OpenTelemetry operator for instrumentation
**Namespace**: control-plane-system
**Status**: Disabled

Manages OpenTelemetry collectors and instrumentation.

### opentelemetry-collector

**Purpose**: Telemetry data collection and export
**Namespace**: control-plane-system
**Status**: Disabled

Collects traces, metrics, and logs for observability platforms.

### datadog-operator

**Purpose**: Datadog monitoring integration
**Namespace**: control-plane-system
**Status**: Disabled

Datadog Agent deployment and configuration.

### cluster-secret

**Purpose**: Secret replication across namespaces
**Namespace**: control-plane-system
**Status**: Disabled

Replicates secrets to multiple namespaces.

### kubecost

**Purpose**: Kubernetes cost monitoring and optimization
**Namespace**: control-plane-system
**Status**: Disabled

Tracks resource costs and provides optimization recommendations.

### observability

**Purpose**: Complete monitoring stack (Prometheus, Grafana, Loki)
**Namespace**: control-plane-system
**Status**: Disabled

Full observability platform including:
- Prometheus for metrics
- Grafana for visualization
- Loki for logs
- Alertmanager for alerts

### backup

**Purpose**: Backup solutions
**Namespace**: backup-system
**Status**: Disabled
**Note**: Not yet configured

Backup and disaster recovery solutions.

### s3-csi-driver

**Purpose**: AWS S3 CSI driver for mounting S3 buckets as volumes
**Namespace**: N/A
**Status**: Addon chart exists but not yet wired into base_chart (no template or values entry)

**Note**: This addon is work-in-progress. The `addon_charts/s3-csi-driver/` directory exists with an upstream dependency on `aws-mountpoint-s3-csi-driver` v1.13.0, but it has no corresponding base_chart template or values.yaml entry.

## Addon Dependencies

Some addons depend on others. Enable in order:

1. **metrics-server** - Required for HPA
2. **cert-manager** - Required for TLS certificates
3. **kube-state-metrics** - Required for full metrics
4. **observability** - Requires metrics-server and kube-state-metrics

## Resource Requirements

Estimated resource usage when enabled:

| Addon | CPU Request | Memory Request | Storage |
|-------|-------------|----------------|---------|
| cert-manager | 100m | 100Mi | - |
| reloader | 10m | 32Mi | - |
| cloudnative-pg | 100m | 100Mi | - |
| azure-service-operator | 200m | 256Mi | - |
| strimzi-operator | 200m | 384Mi | - |
| k6-operator | 100m | 128Mi | - |
| wiremock | 100m | 256Mi | - |
| metrics-server | 100m | 200Mi | - |
| kube-state-metrics | 10m | 32Mi | - |
| observability | 2000m | 4Gi | 50Gi |
| kubecost | 500m | 1Gi | 10Gi |

## Sync Waves

Addons are deployed in order based on sync wave:

| Wave | Addons |
|------|--------|
| 0 | resources, sample |
| 1 | karpenter |
| 2 | metrics-server |
| 3 | providers |
| 4 | kube-state-metrics |
| 5 | node-problem-detector |
| 6 | opentelemetry-operator |
| 7 | opentelemetry-collector |
| 8 | datadog-operator |
| 9 | cert-manager |
| 10 | reloader |
| 11 | providers-config |
| 13 | backup |
| 14 | cluster-secret |
| 15 | kubecost |
| 16 | observability |
| 17 | cloudnative-pg |
| 18 | azure-service-operator |
| 19 | strimzi-operator |
| 20 | k6-operator |
| 21 | wiremock |

## Adding Custom Addons

See [Adding Addons Guide](../guides/adding-addons.md) for instructions.

## Related Documentation

- [Enabling Addons](../guides/enabling-addons.md)
- [Values Schema](values-schema.md)
- [GitOps Workflow](../architecture/gitops-workflow.md)
