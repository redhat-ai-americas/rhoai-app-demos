# AWS GPU MachineSets

GPU-enabled MachineSets for AWS EC2 instances with NVIDIA GPUs.

## Available Instance Types

| Instance Type | GPUs | GPU Memory | vCPUs | RAM | Cost/hr* | Use Case |
|---------------|------|------------|-------|-----|----------|----------|
| g4dn.xlarge | 1x T4 | 16GB | 4 | 16GB | ~$0.53 | Cost-effective, embedding models, small models |
| g6.2xlarge | 1x L4 | 24GB | 8 | 32GB | ~$1.10 | Production, single model inference |
| g6.4xlarge | 1x L4 | 24GB | 16 | 64GB | ~$2.15 | Large models, vision models, high throughput |

*Approximate on-demand pricing (us-east-1, subject to change)

## Prerequisites

- OpenShift cluster on AWS
- AWS quota for GPU instances (g4dn, g6)
- Cluster in region that supports GPU instances
  - g4dn: Available in most regions
  - g6: us-east-1, us-west-2, eu-west-1, and others

## Usage

### Via ArgoCD (Recommended)

```bash
# Deploy g4dn.xlarge (most cost-effective, great for embedding models)
oc apply -f gitops/infra/gpu-machineset-aws-g4dn-xlarge.yaml

# Deploy g6.2xlarge (recommended for most workloads)
oc apply -f gitops/infra/gpu-machineset-aws-g6.yaml

# Deploy g6.4xlarge (for larger models and high throughput)
oc apply -f gitops/infra/gpu-machineset-aws-g6-4xlarge.yaml
```

### Customize Parameters

Edit `params.yaml` in the overlay:

```yaml
instanceType: g4dn.xlarge  # or g6.2xlarge, g6.4xlarge
replicas: 2                # Add more GPU nodes
```

Then apply:

```bash
# For g4dn.xlarge
kustomize build infra/gpu-machineset/aws/overlays/g4dn-xlarge | oc apply -f -

# For g6.2xlarge
kustomize build infra/gpu-machineset/aws/overlays/g6-2xlarge | oc apply -f -

# For g6.4xlarge
kustomize build infra/gpu-machineset/aws/overlays/g6-4xlarge | oc apply -f -
```

### Manual Application

```bash
# Build and apply
kustomize build infra/gpu-machineset/aws/overlays/g6-2xlarge | oc apply -f -

# Monitor
oc get machineset -n openshift-machine-api -w
oc get machine -n openshift-machine-api
oc get nodes -l nvidia.com/gpu.present=true
```

## Cost Considerations

Approximate On-Demand pricing (subject to change):
- **g4dn.xlarge**: ~$0.53/hour (best value for cost-conscious workloads)
- **g6.2xlarge**: ~$1.10/hour (good balance of performance and cost)
- **g6.4xlarge**: ~$2.15/hour (high performance for demanding workloads)

**Cost Savings Tips:**
- Use Spot instances for 60-70% discount
- Scale down when not in use
- Start with g4dn.xlarge for development and testing
- Use g6.2xlarge for production single-model workloads

**Model Recommendations by Instance:**
- **g4dn.xlarge**: Qwen3-VL-Embedding-2B, small 7B models, embedding workloads
- **g6.2xlarge**: Granite 7B, Qwen3-VL-4B, general inference
- **g6.4xlarge**: Qwen3-VL-8B, Llama 3 8B, vision models, high-throughput scenarios

## Troubleshooting

### Machine Stays in "Provisioning"

```bash
oc describe machine <machine-name> -n openshift-machine-api
```

Common causes:
- AWS quota limit reached
- Instance type not available in AZ
- IAM permissions issue

### Node Doesn't Show GPU

```bash
# Check node labels
oc get nodes -L nvidia.com/gpu.present

# Install NVIDIA GPU Operator if needed
```

See main troubleshooting guide for more details.
