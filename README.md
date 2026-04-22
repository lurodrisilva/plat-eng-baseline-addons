Platform Engineering k8s Baseline Addons

[![Helm CI](https://github.com/lurodrisilva/gitops/actions/workflows/helm-ci.yml/badge.svg?branch=master)](https://github.com/lurodrisilva/gitops/actions/workflows/helm-ci.yml)

A GitOps repository for managing AKS cluster addons and platform services using ArgoCD. This repository works in conjunction with [01-aks-tf](../01-aks-tf/) which deploys the infrastructure (AKS, ArgoCD, Crossplane, Azure Service Operators).

## 🚀 Quick Start

This repository is automatically deployed by ArgoCD once the infrastructure is provisioned:

1. **Deploy Infrastructure** (from `01-aks-tf`)
   ```bash
   cd ../01-aks-tf
   make init ENV=dev
   make apply
   ```

2. **ArgoCD Auto-Deploys This Repo**
   - ArgoCD automatically syncs `base_chart/` from this repository
   - Enabled addons are deployed based on `base_chart/values.yaml`

3. **Access ArgoCD**
   ```bash
   # ArgoCD URL
   http://luciano-argocd.eastus.cloudapp.azure.com
   
   # Get admin password
   kubectl -n devops-system get secret argocd-initial-admin-secret \
     -o jsonpath="{.data.password}" | base64 -d
   ```

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Available Addons](#available-addons)
- [Documentation](#documentation)
- [Quick Reference](#quick-reference)

## 🎯 Overview

This GitOps repository manages Kubernetes platform addons through ArgoCD Applications. Each addon is:

- **Declaratively defined** in `base_chart/templates/`
- **Configured** via `base_chart/values.yaml`
- **Automatically deployed** by ArgoCD when enabled
- **Self-healing** with automated sync policies

### Key Features

- **GitOps Workflow**: All changes through Git
- **ArgoCD Applications**: Declarative addon management
- **Helm-based**: Addons use Helm charts with dependencies
- **Namespace Isolation**: Organized by system component
- **Automated Sync**: Self-healing and pruning enabled
- **Sync Waves**: Controlled deployment order

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    AKS Cluster (01-aks-tf)                      │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                  ArgoCD (devops-system)                    │ │
│  │                                                            │ │
│  │  Monitors: https://github.com/lurodrisilva/gitops          │ │
│  │  Path: base_chart/                                         │ │
│  └────────────────────────────────────────────────────────────┘ │
│                           ↓                                     │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │            ArgoCD Applications (from base_chart)           │ │
│  │                                                            │ │
│  │  Each addon in values.yaml (enabled: true) creates:        │ │
│  │  • ArgoCD Application resource                             │ │
│  │  • Points to: addon_charts/<addon_name>/                   │ │
│  │  • Deploys: Helm chart with values                         │ │
│  └────────────────────────────────────────────────────────────┘ │
│                           ↓                                     │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                   Deployed Addons                          │ │
│  │                                                            │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │ │
│  │  │ Cert-Manager │  │   Reloader   │  │CloudNative-PG│      │ │
│  │  │  (enabled)   │  │  (enabled)   │  │  (enabled)   │      │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘      │ │
│  │                                                            │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │     More Addons (as enabled in values.yaml)          │  │ │
│  │  │  Metrics • Observability • Karpenter • etc.          │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### How It Works

1. **Infrastructure Deployment** (`01-aks-tf`):
   - Terraform deploys AKS cluster
   - Installs ArgoCD, Crossplane, Vault
   - Configures ArgoCD to watch this repository

2. **GitOps Application** (this repo):
   - ArgoCD deploys the `base_chart` Helm chart
   - Chart reads `values.yaml` for enabled addons
   - Creates ArgoCD Application for each enabled addon

3. **Addon Deployment**:
   - Each Application points to `addon_charts/<addon_name>/`
   - Helm chart with dependencies deployed
   - Continuous sync and self-healing

## 📁 Project Structure

```
00-aks-baseline/
├── README.md                          # This file
├── docs/                              # Documentation
│   ├── README.md                      # Documentation index
│   ├── setup/
│   │   └── quickstart.md              # Getting started
│   ├── guides/
│   │   ├── adding-addons.md           # How to add new addons
│   │   └── enabling-addons.md         # How to enable/disable addons
│   ├── architecture/
│   │   └── gitops-workflow.md         # GitOps architecture details
│   └── reference/
│       ├── addon-list.md              # Complete addon reference
│       └── values-schema.md           # values.yaml documentation
│
├── base_chart/                        # Main Helm chart (ArgoCD app-of-apps)
│   ├── Chart.yaml                     # Chart metadata
│   ├── values.yaml                    # Addon enable/disable configuration
│   └── templates/                     # ArgoCD Application templates
│       ├── 00-resources.yaml          # Crossplane resources
│       ├── 01-karpenter.yaml          # Karpenter autoscaler
│       ├── 02-metrics-server.yaml     # Metrics server
│       ├── 03-providers.yaml          # Crossplane providers
│       ├── 04-kube-state-metrics.yaml # Kube state metrics
│       ├── 09-cert-manager.yaml       # Certificate manager
│       ├── 10-reloader.yaml           # Config/Secret reloader
│       ├── 14-cloudnative-pg.yaml     # PostgreSQL operator
│       └── ...                        # Other addons
│
└── addon_charts/                      # Individual addon Helm charts
    ├── cert-manager/                  # Certificate management
    │   ├── Chart.yaml                 # Chart with upstream dependency
    │   └── values.yaml                # Custom values
    ├── reloader/                      # Automatic pod reloader
    ├── cloudnative-pg/                # PostgreSQL operator
    ├── karpenter/                     # Cluster autoscaler
    ├── metrics-server/                # Resource metrics
    ├── kube-state-metrics/            # Cluster state metrics
    ├── observability/                 # Monitoring stack
    ├── opentelemetry-operator/        # OpenTelemetry
    ├── opentelemetry-collector/       # Telemetry collector
    ├── providers/                     # Crossplane providers
    ├── providers-config/              # Provider configurations
    ├── resources/                     # Crossplane managed resources
    └── ...                            # Additional addons
```

## 🎛️ Available Addons

### Currently Enabled

| Addon | Purpose | Namespace |
|-------|---------|-----------|
| **cert-manager** | TLS certificate management | control-plane-system |
| **reloader** | Auto-reload pods on ConfigMap/Secret changes | control-plane-system |
| **cloudnative-pg** | PostgreSQL operator for databases | resources-system |
| **azure-service-operator** | Manage Azure resources via Kubernetes CRDs | resources-system |
| **strimzi-operator** | Strimzi Kafka operator | messaging-system |

### Available (Disabled)

| Addon | Purpose | Namespace |
|-------|---------|-----------|
| resources | Crossplane managed Azure resources | resources-system |
| providers | Crossplane provider installations | resources-system |
| providers-config | Crossplane provider configurations | resources-system |
| karpenter | Kubernetes cluster autoscaler | karpenter |
| metrics-server | Core resource metrics | control-plane-system |
| kube-state-metrics | Kubernetes object metrics | control-plane-system |
| node-problem-detector | Node health monitoring | control-plane-system |
| opentelemetry-operator | OpenTelemetry operator | control-plane-system |
| opentelemetry-collector | Telemetry collection | control-plane-system |
| datadog-operator | Datadog monitoring | control-plane-system |
| cluster-secret | Secret replication | control-plane-system |
| kubecost | Cost monitoring | control-plane-system |
| observability | Full monitoring stack | control-plane-system |
| backup | Backup solutions | backup-system |
| k6-operator | Grafana k6 load-testing operator | testing-system |

## 📚 Documentation

### Getting Started
- [**Quickstart Guide**](docs/setup/quickstart.md) - Deploy and configure addons
- [**Enabling Addons**](docs/guides/enabling-addons.md) - How to enable/disable addons

### Guides
- [**Adding New Addons**](docs/guides/adding-addons.md) - Add custom addons
- [**GitOps Workflow**](docs/architecture/gitops-workflow.md) - Understanding the workflow

### Reference
- [**Addon Reference**](docs/reference/addon-list.md) - Complete addon details
- [**Values Schema**](docs/reference/values-schema.md) - Configuration reference

## ⚡ Quick Reference

### Enable an Addon

1. Edit `base_chart/values.yaml`:
   ```yaml
   metrics_server:
     addon_name: metrics-server
     enabled: true              # Change to true
     namespace: control-plane-system
   ```

2. Commit and push:
   ```bash
   git add base_chart/values.yaml
   git commit -m "Enable metrics-server addon"
   git push
   ```

3. ArgoCD auto-syncs within ~3 minutes (or sync manually)

### Check Addon Status

```bash
# List all ArgoCD applications
kubectl get applications -n control-plane-system

# Check specific addon
kubectl get application cert-manager -n control-plane-system

# View addon resources
kubectl get all -n control-plane-system -l app.kubernetes.io/instance=cert-manager
```

### Access ArgoCD UI

```bash
# URL
http://luciano-argocd.eastus.cloudapp.azure.com

# Username
admin

# Password
kubectl -n devops-system get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

### Manual Sync

```bash
# Via kubectl
kubectl patch application cert-manager -n control-plane-system \
  --type merge -p '{"operation":{"sync":{}}}'

# Via ArgoCD CLI
argocd app sync cert-manager
```

## 🔧 Configuration

### Global Settings

Edit `base_chart/values.yaml`:

```yaml
global:
  control_plane:
    namespace: control-plane-system      # ArgoCD namespace
    project: addons-project              # ArgoCD project
    repo: https://github.com/lurodrisilva/gitops  # This repo
    deployment:
      limit: 5                           # Retry limit
      backoff:
        duration: 240s                   # Initial backoff
        factor: 2                        # Backoff multiplier
        maxDuration: 10m                 # Max backoff duration
```

### Addon Configuration

Each addon in `values.yaml`:

```yaml
addon_name:
  addon_name: <name>          # Matches directory in addon_charts/
  enabled: true/false         # Enable/disable addon
  namespace: <namespace>      # Target namespace
```

## 🔗 Related Projects

- [**01-aks-tf**](../01-aks-tf/) - Infrastructure provisioning with Terraform
  - Deploys AKS cluster
  - Installs ArgoCD, Crossplane, Vault
  - Configures Azure infrastructure

## 📖 Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Helm Documentation](https://helm.sh/docs/)
- [GitOps Principles](https://www.gitops.tech/)
- [App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make changes and test locally
4. Submit a pull request

---

**Note**: This repository is designed to work with the infrastructure deployed by [01-aks-tf](../01-aks-tf/). Make sure the infrastructure is deployed first before using this GitOps repository.
