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
- OpenShift 4.19+ with cluster-admin access
- 1x GPU node (AWS g5.xlarge or Azure NC6s_v3)
- 100Gi+ storage
- HuggingFace token (optional for Granite, required for Llama)

**Components**:
- RHOAI operator and DataScienceCluster
- GPU-enabled node
- Granite 7B LLM (or Llama 3 8B)
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

### Common Requirements for All Demos

**Cluster**:
- OpenShift 4.19 or later
- Cluster-admin access
- Sufficient quota for GPU instances and storage

**Client Tools**:
```bash
oc version  # OpenShift CLI
```

**Cloud Provider Access** (for GPU demos):
- AWS: Ability to create g5.xlarge instances
- Azure: Ability to create Standard_NC6s_v3 VMs
- Appropriate quotas and service limits

**HuggingFace Account** (for gated models):
- Create account at https://huggingface.co
- Generate access token with read permissions
- Accept license agreements for gated models (e.g., Llama)

### Demo-Specific Requirements

Check each demo's README for specific prerequisites. For example, the AnythingLLM demo requires:
- 1x GPU node (g5.xlarge or NC6s_v3)
- 100Gi+ storage for model
- 8+ CPU cores, 32Gi+ RAM for model serving


## Getting Started

All demos require OpenShift GitOps and RHOAI as a foundation. Install these first:

### Step 1: Install OpenShift GitOps

```bash
# Step 1a: Install the operator subscription
oc apply -k platform/gitops-operator/base/

# Wait for the operator to install (1-2 minutes)
oc wait --for=condition=Available deployment/openshift-gitops-operator-controller-manager \
  -n openshift-operators --timeout=300s

# Step 1b: Create the ArgoCD instance
oc apply -k platform/gitops-operator/instance/

# Wait for ArgoCD server to be ready (1-2 minutes)
oc wait --for=condition=Ready pod \
  -l app.kubernetes.io/name=openshift-gitops-server \
  -n openshift-gitops --timeout=300s
```

**Access ArgoCD UI** (optional):
```bash
ARGOCD_URL=$(oc get route openshift-gitops-server \
  -n openshift-gitops -o jsonpath='{.spec.host}')
ARGOCD_PASSWORD=$(oc get secret openshift-gitops-cluster \
  -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d)

echo "ArgoCD UI: https://${ARGOCD_URL}"
echo "Username: admin"
echo "Password: ${ARGOCD_PASSWORD}"
```

### Step 2: Install RHOAI

```bash
oc apply -f gitops/platform/rhoai-operator.yaml

# Wait for operator (5-10 minutes)
oc wait --for=condition=Ready pod \
  -l name=rhods-operator \
  -n redhat-ods-operator --timeout=600s

# Verify DataScienceCluster
oc get datasciencecluster -A
```

### Step 3: Follow a Demo Guide

Choose a demo from [Available Demos](#available-demos) and follow its detailed README for:
- GPU deployment
- Storage setup
- Model downloads
- Application configuration
- Demo walkthrough

**Example**: [AnythingLLM RAG Demo →](demos/anythingllm-rag-demo/README.md)


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
Common causes: AWS/Azure quota limits, invalid instance type, IAM permissions

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

**Use smaller instances**: g5.xlarge (~$1/hr) for demos vs larger instances for production

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

