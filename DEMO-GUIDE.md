# RHOAI Demo Guide

Quick reference for available demos, their prerequisites, and getting started.

## Table of Contents

- [Available Demos](#available-demos)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Troubleshooting](#troubleshooting)

## Available Demos

### AnythingLLM RAG Demo

**What**: Complete RAG system with document chat using AnythingLLM and RHOAI

**Use Case**: Upload documents and chat with an AI that can reference and cite them accurately

**Duration**: 30-45 minutes (including model download)

**Difficulty**: Intermediate

**Prerequisites**:
- OpenShift 4.16+ with cluster-admin access
- 1x GPU node (see [GPU deployment guide](README.md#3-deploy-gpu-nodes))
- 100Gi+ storage
- HuggingFace token (required for gated models)

**Components**:
- RHOAI operator and DataScienceCluster
- GPU-enabled node
- Qwen3-VL 8B multimodal model (or Granite 7B, Llama 3 8B)
- AnythingLLM + ChromaDB
- PVC or MinIO storage

**Get Started**: Follow the detailed walkthrough in [demos/anythingllm-rag-demo/README.md](demos/anythingllm-rag-demo/)

---

### n8n Workflow Automation Demo

**Status**: Coming Soon

**What**: Workflow automation with AI-powered nodes using RHOAI backend

**Use Case**: Build automated workflows that leverage AI models for data processing, content generation, and decision-making

---

### LiteLLM Gateway Demo

**Status**: Coming Soon

**What**: Deploy LiteLLM as a unified gateway to multiple RHOAI-hosted models

**Use Case**: Provide a single API endpoint with load balancing and fallback capabilities across multiple models


## Prerequisites

### Platform Prerequisites

All demos require:
- **OpenShift 4.16+** with cluster-admin access (see [main README](README.md#prerequisites) for full requirements)
- **OpenShift GitOps** installed (see [Getting Started](#getting-started))
- **RHOAI** installed (see [Getting Started](#getting-started))

### Demo-Specific Requirements

**Cloud Provider Access** (for GPU demos):
- AWS: Ability to create g6.2xlarge or g6.4xlarge instances
- Appropriate quotas and service limits

**HuggingFace Account** (for gated models):
- Create account at https://huggingface.co
- Generate access token with read permissions
- Accept license agreements for gated models (e.g., Llama)

Check each demo's README for detailed prerequisites. For example, the AnythingLLM demo requires:
- 1x GPU node (g6.2xlarge or NC6s_v3)
- 100Gi+ storage for model
- 8+ CPU cores, 32Gi+ RAM for model serving


## Getting Started

All demos require OpenShift GitOps and RHOAI as a foundation. 

### Prerequisites Setup

**Before starting any demo**, complete the platform setup in the main [README.md](README.md):

1. **[Install OpenShift GitOps](README.md#1-install-openshift-gitops)** - Required for GitOps-based deployment (~3-4 minutes)
2. **[Install RHOAI](README.md#2-install-rhoai)** - Required for model serving (~5-10 minutes)
3. **[Deploy GPU Nodes](README.md#3-deploy-gpu-nodes)** - Required for most demos (~5-10 minutes)

**Verify prerequisites are ready:**
```bash
# Check GitOps is running
oc get pods -n openshift-gitops

# Check RHOAI is ready
oc get datasciencecluster -A

# Check GPU nodes are ready (if required for your demo)
oc get nodes -l nvidia.com/gpu.present=true
```

All should show Running/Ready status before proceeding.

### Choose and Run a Demo

Once prerequisites are complete:

1. Choose a demo from [Available Demos](#available-demos)
2. Follow its detailed README for:
   - Storage setup
   - Model downloads
   - Application configuration
   - Demo walkthrough

**Example**: [AnythingLLM RAG Demo →](demos/anythingllm-rag-demo/README.md)

**Note**: GPU deployment is now covered in the main README prerequisites (Step 3), so individual demos will reference that section rather than duplicating the instructions.


## Troubleshooting

Quick troubleshooting for common issues. For detailed help, see component-specific READMEs.

### Platform Issues

**ArgoCD Application Stuck**:
```bash
oc get application -n openshift-gitops
oc describe application <app-name> -n openshift-gitops
```

**RHOAI Operator Not Installing**:
```bash
oc get subscription -n redhat-ods-operator
oc logs -l name=rhods-operator -n redhat-ods-operator --tail=50
```

**DataScienceCluster Not Ready**:
```bash
oc get datasciencecluster -o yaml
# Check .status.conditions for specific issues
```

### Infrastructure Issues

**GPU Nodes Not Provisioning**:
```bash
oc get machineset -n openshift-machine-api
oc get machine -n openshift-machine-api
oc describe machine <machine-name> -n openshift-machine-api
```
Common causes: AWS quota limits, invalid instance type, IAM permissions, missing cluster-info ConfigMap

See the [GPU deployment troubleshooting guide](infra/gpu-machineset/README.md#troubleshooting) for detailed help.

**GPU Not Detected**:
```bash
oc get nodes -L nvidia.com/gpu.present
oc get pods -n nvidia-gpu-operator  # Check if GPU operator installed
```

**Model Download Job Failing**:
```bash
oc get jobs -n model-downloads
oc logs job/<job-name> -n model-downloads
```
Common causes: Invalid HuggingFace token, insufficient storage, gated model without license acceptance

### Application Issues

**AnythingLLM Not Connecting to RHOAI**:
```bash
oc logs -l app=anythingllm -n anythingllm
oc get inferenceservice -A
# Test model endpoint:
oc run curl-test --image=curlimages/curl -it --rm -- \
  curl http://<inference-service>.<namespace>.svc.cluster.local/v1/models
```

**Application Pod CrashLoopBackOff**:
```bash
oc describe pod <pod-name> -n <namespace>
oc logs <pod-name> -n <namespace> --previous
```
Common causes: SCC issues, missing secrets, insufficient resources

**Cannot Access Routes**:
```bash
oc get route -n <namespace>
oc get route <route-name> -n <namespace> -o yaml | grep -A5 status
```

### Getting More Help

- **Component-specific issues**: See READMEs in respective directories
  - [GPU MachineSets](infra/gpu-machineset/README.md)
  - [Model Downloads](platform/models/README.md)
  - [Secrets Management](secrets/README.md)
- **RHOAI production support**: Contact Red Hat Support
- **Demo issues**: Open an issue in this repository

## Demo Tips

### For Technical Audiences

- Show ArgoCD UI with all Applications deployed
- Explain GitOps pattern and how changes sync from Git
- Demonstrate scaling and GPU utilization
- Walk through the architecture and component interactions

### For Business Audiences

- Focus on the UI and user experience
- Emphasize data sovereignty (everything runs in your cluster)
- Highlight cost savings vs cloud AI APIs
- Discuss compliance and security benefits

### Cost Management

**Scale down when not in use**:
```bash
oc scale machineset <machineset-name> --replicas=0 -n openshift-machine-api
```

**Choose right instance size**: g6.2xlarge (~$1.10/hr) recommended for most workloads, g6.4xlarge (~$2.15/hr) for large models

**Monitor usage**: Set up billing alerts and use cluster autoscaler

### Production Considerations

These demos are for **demonstration purposes**. For production:
- High availability (multiple replicas, PDBs)
- Security (network policies, RBAC, secret management)
- Monitoring (Prometheus, Grafana, alerting)
- Backup & DR
- Resource limits and quotas
- Compliance (image scanning, audit logging)

## Resources

- [Red Hat OpenShift AI Documentation](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai)
- [OpenShift GitOps Documentation](https://docs.openshift.com/gitops/latest/)
- [Application Documentation](https://docs.anythingllm.com/) (AnythingLLM, n8n, LiteLLM)
- [Main README](README.md) - Repository overview and quick start

