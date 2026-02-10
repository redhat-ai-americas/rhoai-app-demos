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

### Via Deployment Script (Recommended)

Use the automated deployment script that handles all cluster configuration:

```bash
# Deploy with default settings (g6.2xlarge, 120GB gp3 volume)
./infra/gpu-machineset/aws/deploy.sh

# Deploy g4dn.xlarge (most cost-effective)
INSTANCE_TYPE=g4dn.xlarge ./infra/gpu-machineset/aws/deploy.sh

# Deploy g6.4xlarge with custom storage
INSTANCE_TYPE=g6.4xlarge \
ROOT_VOLUME_SIZE=200 \
ROOT_VOLUME_TYPE=gp3 \
ROOT_VOLUME_IOPS=5000 \
./infra/gpu-machineset/aws/deploy.sh
```

**Configuration Options:**
- `INSTANCE_TYPE` - Instance type (default: `g6.2xlarge`)
  - `g4dn.xlarge` - Most cost-effective (~$0.53/hr)
  - `g6.2xlarge` - Recommended for most workloads (~$1.10/hr)
  - `g6.4xlarge` - High-performance (~$2.15/hr)
- `ROOT_VOLUME_SIZE` - Root volume size in GB (default: `120`)
- `ROOT_VOLUME_TYPE` - Volume type: `gp3` (recommended) or `gp2` (default: `gp3`)
- `ROOT_VOLUME_IOPS` - IOPS for gp3 volumes (default: `3000`)

The script automatically:
1. Gathers your cluster information (name, region, AZ, AMI)
2. Logs in to ArgoCD
3. Creates the ArgoCD Application
4. Sets cluster-specific Helm parameters
5. Syncs the application to deploy the MachineSet

**MachineSet Naming:**
Each instance type creates a uniquely named MachineSet to avoid conflicts:
- g4dn.xlarge: `{clusterName}-gpu-g4dn-{az}` (e.g., `cluster-abc-gpu-g4dn-us-east-2a`)
- g6.2xlarge: `{clusterName}-gpu-g6-{az}` (e.g., `cluster-abc-gpu-g6-us-east-2a`)
- g6.4xlarge: `{clusterName}-gpu-g6-4x-{az}` (e.g., `cluster-abc-gpu-g6-4x-us-east-2a`)

This allows you to deploy multiple GPU instance types in the same cluster simultaneously.

**Machine and Node Labels:**
Each machine and node gets labeled for easy filtering:
- `gpu-instance-type={instanceType}` - The EC2 instance type (e.g., `g4dn.xlarge`, `g6.2xlarge`)
- `gpu-type-suffix={suffix}` - The short suffix (e.g., `g4dn`, `g6`, `g6-4x`)
- `nvidia.com/gpu.present=true` - Standard GPU node label (added by GPU Operator)

**AWS Instance Tags:**
EC2 instances are tagged for visibility in the AWS console:
- `gpu-node=true` - Marks this as a GPU node
- `gpu-instance-type={instanceType}` - The instance type
- `gpu-type={suffix}` - The short suffix

Then monitor with:
```bash
# Watch MachineSet creation
oc get machineset -n openshift-machine-api -w

# View all GPU machines
oc get machine -n openshift-machine-api -l gpu-node=true

# View machines by instance type
oc get machine -n openshift-machine-api -l gpu-instance-type=g4dn.xlarge
oc get machine -n openshift-machine-api -l gpu-instance-type=g6.2xlarge
oc get machine -n openshift-machine-api -l gpu-instance-type=g6.4xlarge

# Or by suffix
oc get machine -n openshift-machine-api -l gpu-type-suffix=g4dn
oc get machine -n openshift-machine-api -l gpu-type-suffix=g6

# Wait for GPU node to be ready (5-10 minutes)
oc wait --for=condition=Ready nodes -l nvidia.com/gpu.present=true --timeout=600s

# View all GPU nodes
oc get nodes -l nvidia.com/gpu.present=true

# View nodes by instance type
oc get nodes -l gpu-instance-type=g4dn.xlarge
oc get nodes -l gpu-instance-type=g6.2xlarge
```

### Via Kustomize (Advanced)

For direct kustomization without ArgoCD, edit `params.yaml` in the overlay:

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

**Note:** The Kustomize approach requires a `cluster-info-aws` ConfigMap in the `openshift-machine-api` namespace. The deployment script (recommended) handles this automatically.

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

### Error: "Resource not found: REPLACE_ME-gpu-REPLACE_ME"

This error occurs when the GitOps Application is applied without setting the Helm parameters first. 

**Solution:** Use the deployment script which handles this automatically:
```bash
INSTANCE_TYPE=g4dn.xlarge ./infra/gpu-machineset/aws/deploy.sh
```

If you already applied the application manually:
```bash
# Delete the failed application
oc delete application gpu-machineset-aws-g4dn-xlarge -n openshift-gitops

# Then use the deployment script
INSTANCE_TYPE=g4dn.xlarge ./infra/gpu-machineset/aws/deploy.sh
```

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
