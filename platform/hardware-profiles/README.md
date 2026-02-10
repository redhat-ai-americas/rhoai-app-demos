# Hardware Profiles

Hardware profiles define resource constraints and node scheduling for model serving.

## Available Profiles

| Profile | Instance Type | GPU | vCPUs | Memory | Use Case |
|---------|--------------|-----|-------|---------|----------|
| g4dn-xlarge | AWS g4dn.xlarge | NVIDIA T4 (16GB) | 4 | 16 GB | Small models, embeddings |
| g6-4xlarge | AWS g6.4xlarge | NVIDIA L4 (24GB) | 16 | 64 GB | Large models, vision |

## Usage

Hardware profiles are selected when deploying InferenceServices in the RHOAI dashboard.

Deploy via GitOps:

```bash
# g4dn.xlarge (T4)
oc apply -f gitops/platform/hardware-profiles/g4dn-xlarge.yaml

# g6.4xlarge (L4)
oc apply -f gitops/platform/hardware-profiles/g6-4xlarge.yaml
```

Profiles ensure models are scheduled on appropriate GPU nodes with correct tolerations.
