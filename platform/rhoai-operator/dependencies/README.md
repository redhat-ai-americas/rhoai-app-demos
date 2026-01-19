# RHOAI Dependencies

This directory contains the required and optional operator dependencies for Red Hat OpenShift AI (RHOAI) 3.x.

## Required Dependencies

These operators **must** be installed before RHOAI 3.x for proper functionality:

### 1. Node Feature Discovery (NFD) Operator
- **Directory**: `nfd-operator/`
- **Purpose**: Detects and labels hardware features on cluster nodes, essential for GPU node discovery
- **Namespace**: `openshift-nfd`
- **Channel**: `stable`

### 2. Red Hat Build for Kueue Operator
- **Directory**: `kueue-operator/`
- **Purpose**: Provides job queuing and resource management for AI workloads
- **Namespace**: `openshift-kueue`
- **Channel**: `stable`

## Optional Dependencies

These operators provide additional monitoring and support capabilities:

### 3. NVIDIA DCGM Operator (GPU Operator)
- **Directory**: `nvidia-dcgm-operator/`
- **Purpose**: Provides GPU monitoring, health checks, and telemetry for NVIDIA GPUs
- **Namespace**: `nvidia-gpu-operator`
- **Channel**: `stable`
- **Source**: `certified-operators`
- **Note**: Only needed if you're using NVIDIA GPUs and want advanced monitoring/support

## Installation

### Install All Required Dependencies

```bash
oc apply -k platform/rhoai-operator/dependencies/
```

This will install NFD and Kueue operators. To also install the NVIDIA DCGM operator, uncomment it in the `kustomization.yaml` file.

### Install Individual Dependencies

```bash
# NFD Operator
oc apply -k platform/rhoai-operator/dependencies/nfd-operator/

# Kueue Operator
oc apply -k platform/rhoai-operator/dependencies/kueue-operator/

# NVIDIA DCGM Operator (optional)
oc apply -k platform/rhoai-operator/dependencies/nvidia-dcgm-operator/
```

### Wait for Operators to Be Ready

```bash
# Wait for NFD operator
oc wait --for=condition=Ready pod -l app=nfd-master \
  -n openshift-nfd --timeout=300s

# Wait for Kueue operator
oc wait --for=condition=Ready pod -l control-plane=controller-manager \
  -n openshift-kueue --timeout=300s

# Wait for NVIDIA operator (if installed)
oc wait --for=condition=Ready pod -l app=gpu-operator \
  -n nvidia-gpu-operator --timeout=300s
```

## Deployment Order

For a complete RHOAI deployment, follow this order:

1. **Dependencies** (this directory) - Install NFD and Kueue
2. **RHOAI Operator** - Install the RHOAI operator subscription
3. **DataScienceCluster** - Deploy the RHOAI instance

Example:
```bash
# 1. Install dependencies
oc apply -k platform/rhoai-operator/dependencies/

# Wait for dependencies to be ready
sleep 120

# 2. Install RHOAI operator
oc apply -k platform/rhoai-operator/base/

# Wait for RHOAI operator
oc wait --for=condition=Ready pod -l name=rhods-operator \
  -n redhat-ods-operator --timeout=600s

# 3. Deploy DataScienceCluster
oc apply -k platform/rhoai-operator/instance/
```

## Version Notes

- **RHOAI 3.x**: Currently uses the `fast-3.x` subscription channel (will change to `stable` when 3.2 is released)
- These dependency versions are compatible with RHOAI 3.x
- See the main RHOAI operator configuration for channel updates

## Troubleshooting

### Check Operator Installation Status

```bash
# Check subscriptions
oc get subscriptions -n openshift-nfd
oc get subscriptions -n openshift-kueue

# Check operator pods
oc get pods -n openshift-nfd
oc get pods -n openshift-kueue

# Check install plans
oc get installplan -n openshift-nfd
oc get installplan -n openshift-kueue
```

### Common Issues

**NFD not detecting GPUs:**
- Ensure GPU nodes are properly provisioned
- Check NFD daemon set is running: `oc get daemonset -n openshift-nfd`
- Verify node labels: `oc get nodes --show-labels | grep feature.node.kubernetes.io`

**Kueue not available:**
- Verify the operator is installed: `oc get csv -n openshift-kueue`
- Check CRDs are present: `oc get crd | grep kueue`

## Resources

- [Node Feature Discovery Documentation](https://docs.openshift.com/container-platform/latest/hardware_enablement/psap-node-feature-discovery-operator.html)
- [Kueue Documentation](https://kueue.sigs.k8s.io/)
- [NVIDIA GPU Operator Documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html)
