# AWS GPU MachineSet Helm Chart

This Helm chart automatically deploys GPU-enabled MachineSets for AWS with auto-detected cluster configuration.

## Prerequisites

The `cluster-info-aws` ConfigMap must exist in the `openshift-machine-api` namespace before deploying. This is created by running:

```bash
oc apply -f gitops/infra/cluster-info-aws.yaml
```

## How It Works

1. **Auto-Detection**: The chart uses Helm's `lookup` function to fetch cluster configuration from the `cluster-info-aws` ConfigMap at render time
2. **Dynamic Values**: Cluster name, region, availability zone, and infrastructure ID are automatically injected into the MachineSet
3. **ArgoCD Management**: ArgoCD continuously reconciles the MachineSet, ensuring it stays in sync with your desired state

## Available Configurations

### g6.2xlarge (Default)
- 1x NVIDIA L4 (24GB GDDR6)
- 8 vCPUs, 32 GB RAM
- Cost: ~$1.10/hour
- Values file: `values-g6-2xlarge.yaml`

### g6.4xlarge
- 1x NVIDIA L4 (24GB GDDR6)
- 16 vCPUs, 64 GB RAM
- Cost: ~$1.62/hour
- Values file: `values-g6-4xlarge.yaml`

## Deployment via GitOps

```bash
# Deploy g6.2xlarge GPU nodes
oc apply -f gitops/infra/gpu-machineset-aws-g6.yaml

# Deploy g6.4xlarge GPU nodes
oc apply -f gitops/infra/gpu-machineset-aws-g6-4xlarge.yaml
```

## Manual Deployment

```bash
# Install with default values (g6.2xlarge)
helm install gpu-machineset ./infra/gpu-machineset/aws/helm \
  --namespace openshift-machine-api

# Install with custom instance type
helm install gpu-machineset ./infra/gpu-machineset/aws/helm \
  --namespace openshift-machine-api \
  --values ./infra/gpu-machineset/aws/helm/values-g6-4xlarge.yaml

# Override replicas
helm install gpu-machineset ./infra/gpu-machineset/aws/helm \
  --namespace openshift-machine-api \
  --set replicas=2
```

## Customization

Edit the values files to customize:
- `instanceType`: AWS instance type (g6.2xlarge, g6.4xlarge, etc.)
- `replicas`: Number of GPU nodes to deploy
- `rootVolume.size`: Root volume size in GB
- `rootVolume.type`: EBS volume type (gp3, gp2, io1, etc.)
- `labels`: Additional node labels
- `taints`: Node taints for workload isolation

## Verification

```bash
# Check MachineSet
oc get machineset -n openshift-machine-api | grep gpu

# Check Machine provisioning
oc get machine -n openshift-machine-api | grep gpu

# Verify GPU nodes are ready
oc get nodes -l nvidia.com/gpu.present=true
```
