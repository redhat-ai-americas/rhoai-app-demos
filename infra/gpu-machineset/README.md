# GPU MachineSets

Kubernetes MachineSet definitions for deploying GPU-enabled nodes on OpenShift.

## Overview

GPU nodes are required for:
- AI model inference (serving)
- Model training
- GPU-accelerated workloads

This directory provides parameterized MachineSet templates for AWS and Azure with various GPU instance types.

## Structure

```
gpu-machineset/
├── base/                  # Base MachineSet template
├── aws/                   # AWS configurations
│   └── overlays/
│       ├── g6-2xlarge/    # 1x L4, 8 vCPU, 32GB RAM
│       └── g6-4xlarge/    # 1x L4, 16 vCPU, 64GB RAM
└── azure/                 # Azure configurations
    └── overlays/
        ├── nc6s-v3/       # 1x V100, 6 vCPU, 112GB RAM
        └── nc12s-v3/      # 2x V100, 12 vCPU, 224GB RAM
```

## Quick Start

### 1. Choose Your Cloud and Instance Type

See cloud-specific READMEs:
- [AWS GPU instances](aws/README.md)
- [Azure GPU instances](azure/README.md)

### 2. Deploy via ArgoCD (Recommended)

```bash
# AWS g6.2xlarge
oc apply -f gitops/infra/gpu-machineset-aws-g6.yaml

# Azure NC6s v3
oc apply -f gitops/infra/gpu-machineset-azure-nc.yaml
```

### 3. Verify Deployment

```bash
# Watch MachineSet creation
oc get machineset -n openshift-machine-api -w

# Wait for Machine to provision (5-10 minutes)
oc get machine -n openshift-machine-api

# Verify GPU node is Ready
oc get nodes -l nvidia.com/gpu.present=true

# Check GPU detection
oc describe node <gpu-node-name> | grep -i gpu
```

## Customization

Each overlay has a `params.yaml` file for easy customization:

```yaml
# Example: infra/gpu-machineset/aws/overlays/g6-2xlarge/params.yaml
instanceType: g6.2xlarge
replicas: 2              # Increase GPU node count
availabilityZone: us-west-2b  # Change AZ
```

After editing, apply:

```bash
kustomize build infra/gpu-machineset/aws/overlays/g6-2xlarge | oc apply -f -
```

## GPU Node Configuration

All GPU nodes are configured with:

**Labels:**
- `node-role.kubernetes.io/gpu=""`
- `nvidia.com/gpu.present="true"`

**Taints:**
- `nvidia.com/gpu=true:NoSchedule`

This ensures only GPU workloads schedule on GPU nodes.

## Scheduling Workloads on GPU Nodes

To schedule a pod on GPU nodes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  nodeSelector:
    nvidia.com/gpu.present: "true"
  tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
  containers:
    - name: app
      image: myapp:latest
      resources:
        limits:
          nvidia.com/gpu: 1  # Request 1 GPU
```

## NVIDIA GPU Operator

For GPU support on OpenShift, you may need the NVIDIA GPU Operator:

```bash
# Install via OperatorHub in OpenShift Console
# Or via CLI:
oc create -f https://raw.githubusercontent.com/NVIDIA/gpu-operator/master/deployments/gpu-operator/bundle.yaml
```

The operator provides:
- GPU drivers
- CUDA runtime
- GPU device plugin
- GPU monitoring

## Cost Management

### Best Practices

1. **Scale Down When Not in Use**
   ```bash
   oc scale machineset <machineset-name> --replicas=0 -n openshift-machine-api
   ```

2. **Choose Right Instance Size**
   - g6.2xlarge ($1.10/hr) - recommended for most workloads
   - g6.4xlarge ($2.15/hr) - for large models and high throughput

3. **Consider Spot Instances**
   - 60-70% cost savings
   - Good for non-critical workloads

4. **Monitor Usage**
   - Set up billing alerts
   - Use cluster autoscaler for dynamic scaling

### Cost Comparison

| Cloud | Instance | GPUs | $/hour (approx) | Best For |
|-------|----------|------|-----------------|----------|
| AWS | g6.2xlarge | 1x L4 | $1.10 | Production, single model |
| AWS | g6.4xlarge | 1x L4 | $2.15 | Large models, vision |
| Azure | NC6s_v3 | 1x V100 | $3.00 | Training |
| Azure | NC12s_v3 | 2x V100 | $6.00 | Multi-GPU |

*Prices are approximate On-Demand rates and vary by region*

## Troubleshooting

### MachineSet Created but No Machines

```bash
oc describe machineset <name> -n openshift-machine-api
```

Check for:
- Invalid instance type
- AZ/region mismatch
- Missing IAM permissions

### Machine Provisioning Failed

```bash
oc describe machine <name> -n openshift-machine-api
```

Common causes:
- **AWS:** Instance quota exceeded, spot unavailable
- **Azure:** VM size not available in region
- **Both:** Network/subnet issues

### Node Ready but GPU Not Detected

```bash
# Check if NVIDIA GPU Operator is installed
oc get pods -n nvidia-gpu-operator

# If not, install it
# Then wait for GPU operator pods to be ready
```

### Pod Pending - "No nodes available"

Check:
1. GPU node is Ready: `oc get nodes`
2. Node has GPU label: `oc get nodes -L nvidia.com/gpu.present`
3. Pod has correct toleration for taint
4. Pod requested GPU resource: `nvidia.com/gpu: 1`

## Adding New Cloud Providers or Instance Types

### To add a new instance type:

1. Create overlay directory:
   ```bash
   mkdir -p infra/gpu-machineset/<cloud>/overlays/<instance-type>
   ```

2. Create `params.yaml` with instance details

3. Create `kustomization.yaml` with patches

4. Test:
   ```bash
   kustomize build infra/gpu-machineset/<cloud>/overlays/<instance-type> | oc apply --dry-run=client -f -
   ```

5. Create ArgoCD Application in `gitops/infra/`

## References

- [OpenShift Machine Management](https://docs.openshift.com/container-platform/latest/machine_management/index.html)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/overview.html)
- [AWS EC2 G5 Instances](https://aws.amazon.com/ec2/instance-types/g5/)
- [Azure NCv3 Series](https://docs.microsoft.com/en-us/azure/virtual-machines/ncv3-series)
