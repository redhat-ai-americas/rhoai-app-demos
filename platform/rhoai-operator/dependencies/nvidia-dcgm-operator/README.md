# NVIDIA GPU Operator

The NVIDIA GPU Operator enables GPU workload scheduling on OpenShift by deploying and managing GPU drivers, device plugins, and monitoring tools.

## Overview

The GPU operator provides:
- **GPU drivers** - NVIDIA kernel modules
- **CUDA runtime** - GPU computing platform
- **GPU device plugin** - Exposes GPUs to Kubernetes scheduler
- **DCGM** - GPU monitoring and telemetry
- **Node Feature Discovery** - Detects GPU hardware

## Deployment

Deploy via GitOps:

```bash
oc apply -f gitops/platform/nvidia-gpu-operator.yaml
```

This creates an ArgoCD Application that deploys:
- Namespace: `nvidia-gpu-operator`
- OperatorGroup
- Subscription (from certified-operators catalog)
- ClusterPolicy with GPU node tolerations

## Verification

Check operator installation:

```bash
# Verify CSV is installed
oc get csv -n nvidia-gpu-operator

# Check operator pods
oc get pods -n nvidia-gpu-operator

# Verify ClusterPolicy
oc get clusterpolicy -n nvidia-gpu-operator
```

Once GPU nodes are available, verify GPU resources:

```bash
# Check GPU nodes show allocatable GPUs
oc get nodes -l nvidia.com/gpu.present=true -o json | jq '.items[].status.allocatable'

# Should show:
# {
#   "nvidia.com/gpu": "1",
#   ...
# }
```

## Configuration

The ClusterPolicy is configured with tolerations to deploy on GPU nodes:

```yaml
spec:
  daemonsets:
    tolerations:
      - effect: NoSchedule
        operator: Exists
        key: nvidia.com/gpu
```

This matches the taints on GPU MachineSets.

## Architecture

```
NVIDIA GPU Operator
├── gpu-operator (deployment)
├── nvidia-driver-daemonset (on GPU nodes)
├── nvidia-container-toolkit-daemonset (on GPU nodes)
├── nvidia-device-plugin-daemonset (on GPU nodes)
├── nvidia-dcgm (daemonset for monitoring)
└── nvidia-dcgm-exporter (daemonset for Prometheus metrics)
```

## Troubleshooting

### Operator not installing

```bash
oc describe subscription gpu-operator-certified -n nvidia-gpu-operator
```

Check for:
- Catalog source availability
- InstallPlan status

### ClusterPolicy not creating resources

```bash
oc describe clusterpolicy gpu-cluster-policy -n nvidia-gpu-operator
```

### GPU not showing as allocatable on nodes

```bash
# Check device plugin logs
oc logs -n nvidia-gpu-operator -l app=nvidia-device-plugin-daemonset

# Check driver logs
oc logs -n nvidia-gpu-operator -l app=nvidia-driver-daemonset
```

Common issues:
- Driver daemonset waiting for GPU node (normal if no GPU nodes exist yet)
- Kernel module compilation in progress
- GPU node doesn't have proper labels/taints

## References

- [NVIDIA GPU Operator Documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/overview.html)
- [OpenShift GPU Support](https://docs.openshift.com/container-platform/latest/hardware_enablement/psap-node-feature-discovery-operator.html)
