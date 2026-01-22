# AWS GPU MachineSet Helm Chart

This Helm chart deploys GPU-enabled MachineSets for AWS with cluster-specific configuration values.

## Prerequisites

### Install ArgoCD CLI

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd

# Or download from: https://github.com/argoproj/argo-cd/releases
```

### Login to ArgoCD

```bash
# Get ArgoCD admin password and server URL
ARGOCD_PASSWORD=$(oc get secret/openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d)
ARGOCD_SERVER=$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}')

# Login
argocd login $ARGOCD_SERVER --username admin --password $ARGOCD_PASSWORD --insecure
```

## How It Works

1. Query OpenShift for cluster configuration
2. Use ArgoCD CLI to set Helm parameters
3. Sync to create the MachineSet

## Deployment via Notebook (Recommended)

Use `platform-deployment.ipynb` which automatically handles cluster detection and ArgoCD CLI setup.

## Manual Deployment

```bash
# Extract cluster information
CLUSTER_NAME=$(oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}')
REGION=$(oc get infrastructure cluster -o jsonpath='{.status.platformStatus.aws.region}')
AVAILABILITY_ZONE=$(oc get machines -n openshift-machine-api -l machine.openshift.io/cluster-api-machine-role=worker -o jsonpath='{.items[0].spec.providerSpec.value.placement.availabilityZone}')
INFRA_ID=$CLUSTER_NAME

# Create Application (without auto-sync to avoid premature deployment)
oc apply -f gitops/infra/gpu-machineset-aws-g6.yaml

# Set Helm parameters
argocd app set gpu-machineset-aws-g6 \
  -p clusterName="$CLUSTER_NAME" \
  -p region="$REGION" \
  -p availabilityZone="$AVAILABILITY_ZONE" \
  -p infraID="$INFRA_ID"

# Enable auto-sync and sync
argocd app set gpu-machineset-aws-g6 --sync-policy automated --auto-prune --self-heal
argocd app sync gpu-machineset-aws-g6
```

## Available Configurations

- **g6.2xlarge**: 1x NVIDIA L4, 8 vCPU, 32GB RAM (~$1.10/hr)
- **g6.4xlarge**: 1x NVIDIA L4, 16 vCPU, 64GB RAM (~$1.62/hr)

Use `gitops/infra/gpu-machineset-aws-g6.yaml` or `gitops/infra/gpu-machineset-aws-g6-4xlarge.yaml`

## Customization

Override default values:

```bash
argocd app set gpu-machineset-aws-g6 -p replicas=2
```

## Verification

```bash
# Check MachineSet
oc get machineset -n openshift-machine-api | grep gpu

# Check GPU nodes
oc get nodes -l nvidia.com/gpu.present=true
```

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
