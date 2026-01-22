# AWS GPU MachineSets

GPU-enabled MachineSets for AWS EC2 g6 instances with NVIDIA L4 GPUs.

## Available Instance Types

| Instance Type | GPUs | GPU Memory | vCPUs | RAM | Use Case |
|---------------|------|------------|-------|-----|----------|
| g6.2xlarge | 1x L4 | 24GB | 8 | 32GB | Production, single model inference |
| g6.4xlarge | 1x L4 | 24GB | 16 | 64GB | Large models, vision models, high throughput |

## Prerequisites

- OpenShift cluster on AWS
- AWS quota for g6 instances
- Cluster in region that supports g6 instances (us-east-1, us-west-2, etc.)

## Usage

### Via ArgoCD

```bash
# Deploy g6.2xlarge (recommended for most workloads)
oc apply -f gitops/infra/gpu-machineset-aws-g6.yaml
```

### Customize Parameters

Edit `params.yaml` in the overlay:

```yaml
instanceType: g6.2xlarge
replicas: 2              # Add more GPU nodes
availabilityZone: us-west-2a  # Your AZ
```

Then apply:

```bash
kustomize build infra/gpu-machineset/aws/overlays/g6-2xlarge | oc apply -f -
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
- g6.2xlarge: ~$1.10/hour
- g6.4xlarge: ~$2.15/hour

Consider:
- Spot instances for cost savings (60-70% discount)
- Scaling down when not in use
- g6.2xlarge is recommended for most demos and single-model workloads

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
