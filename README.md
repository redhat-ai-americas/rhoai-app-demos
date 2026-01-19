# RHOAI Application Demos

This is a modular, GitOps-enabled repository for deploying and demonstrating Red Hat OpenShift AI (RHOAI) with various AI applications.

The value of AI is seen most easily through the use of AI applications that users can interact with, and there are a number of really good out-of-the-box, production-ready, AI applications. 

Pairing these applications with a RHOAI platform serves the following purposes:
- Illustrates the value of RHOAI as an AI platform in-context
- Illustrates solutions for common AI use-cases
- Makes it possible to create demos that resonate with multiple user types

## Overview

Red Hat OpenShift AI (RHOAI) is an enterprise AI/ML platform. This repository provides everything you need to:

- **Deploy RHOAI** with GitOps (ArgoCD)
- **Run AI applications** like AnythingLLM, n8n, and LiteLLM
- **Demonstrate end-to-end use cases** with step-by-step guides
- **Mix and match components** using a modular architecture

### Key Features

- **GitOps-based** - All components deployed via ArgoCD
- **Modular** - Pick only what you need from the catalog
- **Multi-cloud** - GPU support for AWS and Azure
- **Configurable** - Parameterized with Kustomize and Helm

## Prerequisites

- **OpenShift 4.16+** with cluster-admin access (4.16+ required for RHOAI 3.x)
- **`oc` CLI** installed and configured
- **Cloud provider access** (for GPU nodes)

## Quick Start

Get started in 3 steps:

### 1. Install OpenShift GitOps

**Important**: The GitOps operator must be installed directly (not via ArgoCD) since ArgoCD doesn't exist yet.

The installation requires two steps to avoid timing issues:

```bash
# Step 1: Install the operator subscription
oc apply -k platform/gitops-operator/base/

# Wait for the operator to install (1-2 minutes)
oc wait --for=condition=Available deployment/openshift-gitops-operator-controller-manager \
  -n openshift-operators --timeout=300s

# Step 2: Create the ArgoCD instance
oc apply -k platform/gitops-operator/instance/

# Wait for ArgoCD server to be ready (1-2 minutes)
oc wait --for=condition=Ready pod -l app.kubernetes.io/name=openshift-gitops-server \
  -n openshift-gitops --timeout=300s
```

### 2. Install RHOAI

RHOAI 3.x requires dependencies to be installed first. The GitOps Application will handle this automatically:

```bash
# This deploys:
# - Node Feature Discovery (NFD) operator
# - Red Hat Build for Kueue operator  
# - RHOAI operator subscription (fast-3.x channel)
# - DataScienceCluster instance
oc apply -f gitops/platform/rhoai-operator.yaml

# Wait for RHOAI (5-10 minutes)
oc wait --for=condition=Ready pod -l name=rhods-operator \
  -n redhat-ods-operator --timeout=600s
```

**Note**: RHOAI 3.x currently uses the `fast-3.x` subscription channel. This will change to `stable` when RHOAI 3.2 is released. See [platform/rhoai-operator/README.md](platform/rhoai-operator/README.md) for configuration details.

### 3. Choose a Demo

See **[DEMO-GUIDE.md](DEMO-GUIDE.md)** for available demos with complete deployment instructions.

Example: [AnythingLLM RAG Demo](demos/anythingllm-rag-demo/README.md) - Document chat with RHOAI backend (30-45 min)

## RHOAI 3.x Information

This repository is configured for **Red Hat OpenShift AI (RHOAI) 3.x**, which includes several important requirements:

### Subscription Channel

RHOAI 3.x is currently deployed using the **`fast-3.x`** subscription channel for early access to 3.x features. When RHOAI 3.2 is released, the recommended channel will be **`stable`**.

The channel can be configured in [`platform/rhoai-operator/base/kustomization.yaml`](platform/rhoai-operator/base/kustomization.yaml).

### Required Dependencies

RHOAI 3.x requires these operators to be installed before deployment:

1. **Node Feature Discovery (NFD) Operator** - Detects hardware features, essential for GPU discovery
2. **Red Hat Build for Kueue Operator** - Provides job queuing and resource management

### Optional Dependencies

For enhanced GPU support:

3. **NVIDIA DCGM Operator** - GPU monitoring and health checks (recommended for NVIDIA GPU clusters)

All dependencies are automatically installed when using the GitOps Application. For manual installation or more details, see [platform/rhoai-operator/README.md](platform/rhoai-operator/README.md).

## Repository Structure

```
rhoai-app-demos/
├── gitops/              # ArgoCD Application catalog - APPLY THESE
│   ├── platform/        # GitOps, RHOAI, model downloads
│   ├── infra/          # GPU nodes, storage
│   └── apps/           # AnythingLLM, n8n, LiteLLM
├── platform/           # Platform component manifests
├── infra/              # Infrastructure component manifests
├── apps/               # Application component manifests
├── demos/              # Demo walkthroughs
└── secrets/            # Secret management docs
```

### How It Works

This repository uses a **GitOps catalog pattern**:

1. **Browse** the catalog in `gitops/` to find components you need
2. **Apply** ArgoCD Application manifests: `oc apply -f gitops/platform/rhoai-operator.yaml`
3. **ArgoCD deploys** the actual manifests from `platform/`, `infra/`, or `apps/` directories
4. **GitOps keeps in sync** - changes in Git automatically sync to cluster

Components are organized in layers:
- **Platform**: Operators and models (deploy first)
- **Infrastructure**: GPU nodes and storage (deploy second)
- **Applications**: AI apps that use RHOAI (deploy last)

See [gitops/README.md](gitops/README.md) for the complete catalog.

## Configuration Examples

Quick examples for common customizations. See component READMEs for details.

### GPU Instance Types

```bash
# Edit parameters for your cloud provider
vim infra/gpu-machineset/aws/overlays/g5-xlarge/params.yaml

# Then apply via GitOps
oc apply -f gitops/infra/gpu-machineset-aws-g5.yaml
```

### Storage Options

```bash
# PVC storage (simpler)
oc apply -f gitops/infra/storage-pvc.yaml

# MinIO storage (S3-compatible)
oc apply -f gitops/infra/storage-minio.yaml
```

### Model Selection

```bash
# Granite 7B
oc apply -f gitops/platform/models/granite-7b-pvc.yaml

# Llama 3 8B
oc apply -f gitops/platform/models/llama-3-8b-pvc.yaml
```

See component-specific READMEs for detailed configuration options:
- [GPU MachineSet Configuration](infra/gpu-machineset/README.md)
- [Model Downloads](platform/models/README.md)
- [Secrets Management](secrets/README.md)

## Contributing

To add a new component:

1. Create manifests in `apps/3rd-party-apps/<app-name>/`
2. Create ArgoCD Application in `gitops/apps/<app-name>.yaml`
3. Add demo guide in `demos/<demo-name>/`
4. Update documentation

See the AnythingLLM implementation as a reference.

## Resources

- [Red Hat OpenShift AI Documentation](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai)
- [OpenShift GitOps Documentation](https://docs.openshift.com/gitops/latest/)
- [Demo Guide](DEMO-GUIDE.md) - Available demos and troubleshooting

## License & Support

This is a demo repository maintained by Red Hat AI Americas. For RHOAI production support, contact Red Hat Support.

