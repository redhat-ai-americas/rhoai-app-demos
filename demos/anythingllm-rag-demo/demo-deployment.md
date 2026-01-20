# AnythingLLM RAG Demo - Deployment Instructions

Simple step-by-step instructions to deploy the AnythingLLM RAG demo.

## Prerequisites

Before starting, ensure you have completed:
- OpenShift GitOps installation (see main README)
- RHOAI Operator installation (see main README)

Verify:
```bash
oc get pods -n openshift-gitops
oc get datasciencecluster -A
```

## Deployment Steps

### 1. Deploy GPU Node (5-10 min)

**⚠️ Configure parameters first:**

```bash
# Find your cluster's availability zones
oc get machines -n openshift-machine-api -o jsonpath='{.items[0].spec.providerSpec.value.placement.availabilityZone}{"\n"}'
```

```bash
# For AWS - Edit parameters to match your cluster
vim infra/gpu-machineset/aws/overlays/g6-2xlarge/params.yaml

# REQUIRED: Update availabilityZone to match your cluster
# OPTIONAL: Adjust replicas or instanceType as needed
```


**Then deploy:**

```bash
# For AWS:
oc apply -f gitops/infra/gpu-machineset-aws-g6.yaml

# For Azure:
# oc apply -f gitops/infra/gpu-machineset-azure-nc.yaml

# Wait for node to be Ready
oc get nodes -l nvidia.com/gpu.present=true -w
```

**✓ Verify:** One GPU node shows `Ready` status

---

### 2. Download Model (10-30 min)

This step creates a PVC and downloads the model to it in the `model-downloads` namespace.

```bash
# Create HuggingFace token secret
read -sp "Enter HuggingFace token: " HF_TOKEN
echo

oc create namespace model-downloads --dry-run=client -o yaml | oc apply -f -
oc create secret generic huggingface-token \
  --from-literal=token=${HF_TOKEN} \
  -n model-downloads

# Start model download (creates PVC and downloads model)
oc apply -f gitops/platform/models/qwen3-vl-8b-pvc.yaml

# Monitor progress
oc logs -f job/download-qwen3-vl-8b -n model-downloads

# Verify PVC was created and is bound
oc get pvc -n model-downloads
```

**✓ Verify:** 
- Job shows `COMPLETIONS: 1/1`
- PVC `qwen3-vl-8b-model-storage` shows `Bound` status

---

### 3. Deploy Model Serving (3-5 min)

Deploy the InferenceService in the `model-downloads` namespace (same as the PVC):

```bash
cat <<EOF | oc apply -f -
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: qwen3-vl-8b
  namespace: model-downloads
spec:
  predictor:
    model:
      modelFormat:
        name: pytorch
      runtime: vllm-runtime
      storageUri: pvc://qwen3-vl-8b-model-storage/Qwen3-VL-8B-Instruct
      resources:
        limits:
          nvidia.com/gpu: "1"
        requests:
          nvidia.com/gpu: "1"
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
EOF

# Wait for model to be ready
oc wait --for=condition=Ready inferenceservice/qwen3-vl-8b \
  -n model-downloads --timeout=600s
```

**✓ Verify:** InferenceService shows `READY=True`

**Test the model:**
```bash
oc run curl-test --image=curlimages/curl -it --rm -n model-downloads -- \
  curl -X POST http://qwen3-vl-8b-predictor.model-downloads.svc.cluster.local/v1/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen3-VL-8B-Instruct", "prompt": "Hello", "max_tokens": 50}'
```

---

### 4. Configure AnythingLLM

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

### 5. Deploy AnythingLLM (2-3 min)

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
# Check all components
oc get nodes -l nvidia.com/gpu.present=true
oc get pvc -n model-downloads
oc get inferenceservice -n model-downloads
oc get pods -n anythingllm

# All should be Running/Ready
```

## Time Estimate

| Step | Time | Notes |
|------|------|-------|
| GPU Node | 5-10 min | Cloud provider provisioning |
| Model Download | 10-30 min | Depends on network speed (includes PVC creation) |
| Model Serving | 3-5 min | Model loading time |
| Configure | < 1 min | Quick file edit |
| AnythingLLM | 2-3 min | Container startup |

**Total: ~25-45 minutes** (mostly model download time)

## Cleanup

Remove all demo components:

```bash
# Delete in reverse order
oc delete -f gitops/apps/anythingllm.yaml
oc delete inferenceservice qwen3-vl-8b -n model-downloads
oc delete -f gitops/platform/models/qwen3-vl-8b-pvc.yaml
oc delete -f gitops/infra/gpu-machineset-aws-g6.yaml

# Delete namespaces
oc delete project anythingllm model-downloads
```

## Troubleshooting

**GPU node not appearing:**
```bash
oc get machine -n openshift-machine-api
oc describe machineset <name> -n openshift-machine-api
```

**Model download fails:**
```bash
oc logs job/download-qwen3-vl-8b -n model-downloads
# Check for: invalid token, network issues, or OOM
```

**InferenceService not ready:**
```bash
oc get pod -n model-downloads
oc logs -l serving.kserve.io/inferenceservice=qwen3-vl-8b -n model-downloads
# Check for: GPU not found, model loading errors, PVC mount issues
```

**AnythingLLM connection error:**
```bash
# Test from AnythingLLM pod
oc exec -it deploy/anythingllm -n anythingllm -- \
  curl http://qwen3-vl-8b-predictor.model-downloads.svc.cluster.local/v1/models
```

## Next Steps

After successful deployment, see [demo-user-instructions.md](./demo-user-instructions.md) to run through the user experience.
