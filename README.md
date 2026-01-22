# RHOAI Application Demos

## WORK-IN-PROGRESS - THIS REPO IS STILL BEING DEVELOPED AND IS NOT COMPLETE

This is a modular, GitOps-enabled repository for deploying and demonstrating Red Hat OpenShift AI (RHOAI) with various AI applications.

The value of AI is seen most easily through the use of AI applications that users can interact with, and there are a number of really good out-of-the-box, open-source, production-ready, AI applications out there. 

Pairing these applications with a RHOAI platform serves the following purposes:
- Illustrates the value of RHOAI as an AI platform in the context of supporting excellent applications
- Fascilitates illustration of solutions for common AI use-cases
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
- **Multi-cloud** - GPU support for AWS (Azure support coming soon)
- **Configurable** - Parameterized with Kustomize and Helm

## Prerequisites

- **OpenShift 4.16+** with cluster-admin access (4.16+ required for RHOAI 3.x)
- **`oc` CLI** installed and configured
- **Cloud provider access** (for GPU nodes)
- **[uv](https://docs.astral.sh/uv/)** (Python package manager) for notebook execution

### Setting Up the Python Environment

This repository uses `uv` for Python dependency management. To set up the environment for running the deployment notebook:

```bash
# Install uv (if not already installed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Sync dependencies and activate the virtual environment
uv sync

# Activate the virtual environment
source .venv/bin/activate  # On macOS/Linux
# OR
.venv\Scripts\activate     # On Windows
```

After activation, you can use an IDE like VSCode (or run Jupyter) to execute the `platform-deployment.ipynb` notebook.

## Deployment

See **[platform-deployment.ipynb](platform-deployment.ipynb)** for step-by-step deployment commands:

1. Install OpenShift GitOps (ArgoCD)
2. Install RHOAI 3.x with dependencies (NFD, Kueue)
3. Deploy GPU nodes (AWS)
4. Download and deploy models

After deployment, see **[DEMO-GUIDE.md](DEMO-GUIDE.md)** for available demos.

## Available Models

| Model | Size | HuggingFace Token | Notes |
|-------|------|-------------------|-------|
| Qwen3-VL 8B | ~18GB | Optional | Multimodal, recommended for RAG demos |
| Granite 7B | ~14GB | Optional | IBM's open instruction model |
| Llama 3 8B | ~16GB | Required | Requires license acceptance on HuggingFace |

## Common Operations

**GPU Node Management:**
```bash
# Scale down GPU nodes when not in use
oc scale machineset <machineset-name> --replicas=0 -n openshift-machine-api

# Scale back up when needed
oc scale machineset <machineset-name> --replicas=1 -n openshift-machine-api
```

**Model Management:**
```bash
# Stop a model (releases GPU)
oc patch inferenceservice <model-name> -n demo \
  --type=merge -p '{"metadata":{"annotations":{"serving.kserve.io/stop":"true"}}}'

# Start a model
oc patch inferenceservice <model-name> -n demo \
  --type=merge -p '{"metadata":{"annotations":{"serving.kserve.io/stop":"false"}}}'

# List inference services
oc get inferenceservice -n demo
```

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

## Additional Information

- **RHOAI Version**: This repo uses RHOAI 3.x with the `fast-3.x` subscription channel
- **GPU Support**: AWS g6.2xlarge (NVIDIA L4) by default. See [infra/gpu-machineset/README.md](infra/gpu-machineset/README.md) for other instance types
- **Dependencies**: NFD and Kueue operators are automatically installed with RHOAI
- **Configuration**: See component-specific READMEs for customization options

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

