# AnythingLLM RAG Demo - Deployment Instructions

Simple step-by-step instructions to deploy the AnythingLLM RAG demo.

## Prerequisites

Before starting, ensure you have completed the main README steps:
- [Step 1: Install OpenShift GitOps](../../README.md#1-install-openshift-gitops)
- [Step 2: Install RHOAI](../../README.md#2-install-rhoai)
- [Step 3: Deploy GPU Nodes](../../README.md#3-deploy-gpu-nodes-aws)
- [Step 4: Download and Deploy Models](../../README.md#4-download-and-deploy-models)

Verify:
```bash
oc get pods -n openshift-gitops
oc get datasciencecluster -A
oc get nodes -l nvidia.com/gpu.present=true
oc get inferenceservice -n model-downloads
```

## Deployment Steps

### 1. Deploy GPU Node and Model

**Prerequisites:**
- GPU node deployed (see [README - Step 3](../../README.md#3-deploy-gpu-nodes-aws))
- Model downloaded and serving (see [README - Step 4](../../README.md#4-download-and-deploy-models))

**Required model:** Qwen3-VL 8B (recommended) or any compatible model

**✓ Verify prerequisites:**

```bash
# GPU node is Ready
oc get nodes -l nvidia.com/gpu.present=true

# Model PVC is Bound
oc get pvc -n model-downloads

# InferenceService is Ready
oc get inferenceservice -n model-downloads
```

---

### 2. Configure AnythingLLM

Update the AnythingLLM configuration to point to the model:

```bash
# Edit the values file
vim apps/3rd-party-apps/anythingllm/helm/values-rhoai.yaml
```

Update these values:
```yaml
llm:
  baseUrl: "http://qwen3-vl-8b-predictor.model-downloads.svc.cluster.local/v1"
  model: "Qwen3-VL-8B-Instruct"

embedding:
  baseUrl: "http://qwen3-vl-8b-predictor.model-downloads.svc.cluster.local/v1"
  model: "Qwen3-VL-8B-Instruct"
```

---

### 3. Deploy AnythingLLM (2-3 min)

```bash
oc apply -f gitops/apps/anythingllm.yaml

# Wait for deployment
oc wait --for=condition=Ready pod \
  -l app=anythingllm \
  -n anythingllm --timeout=300s

# Get the URL
oc get route anythingllm -n anythingllm -o jsonpath='https://{.spec.host}{"\n"}'
```

**✓ Verify:** AnythingLLM URL is accessible in browser

---

## Quick Verification

```bash
# Check AnythingLLM deployment
oc get pods -n anythingllm
oc get route anythingllm -n anythingllm

# Test model connectivity from AnythingLLM
oc exec -it deploy/anythingllm -n anythingllm -- \
  curl http://qwen3-vl-8b-predictor.model-downloads.svc.cluster.local/v1/models
```

## Time Estimate

| Step | Time | Notes |
|------|------|-------|
| Prerequisites | 20-45 min | GPU node + model (see main README) |
| Configure AnythingLLM | < 1 min | Quick file edit |
| Deploy AnythingLLM | 2-3 min | Container startup |

**Total: ~3-5 minutes** (with prerequisites already complete)

**Total from scratch: ~25-50 minutes** (including all prerequisites)

## Cleanup

Remove AnythingLLM:

```bash
oc delete -f gitops/apps/anythingllm.yaml
oc delete project anythingllm
```

To clean up GPU node and model resources, see the main [README](../../README.md).

## Troubleshooting

**For GPU node or model issues:** See main [README](../../README.md) troubleshooting sections.

**AnythingLLM connection error:**
```bash
# Test from AnythingLLM pod
oc exec -it deploy/anythingllm -n anythingllm -- \
  curl http://qwen3-vl-8b-predictor.model-downloads.svc.cluster.local/v1/models
```

**AnythingLLM pod not starting:**
```bash
oc get pods -n anythingllm
oc logs -l app=anythingllm -n anythingllm
oc describe pod -l app=anythingllm -n anythingllm
```

## Next Steps

After successful deployment, see [demo-user-instructions.md](./demo-user-instructions.md) to run through the user experience.
