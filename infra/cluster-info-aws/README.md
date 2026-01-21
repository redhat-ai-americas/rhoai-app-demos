# AWS Cluster Info Detection

This component automatically detects AWS-specific cluster configuration and creates a ConfigMap that other components can reference.

## What It Does

The cluster-info detection job queries your OpenShift cluster to extract:
- **Cluster Name** - Used to name GPU MachineSets correctly
- **Infrastructure Name** - OpenShift infrastructure identifier
- **Platform Type** - Validates this is an AWS cluster
- **Region** - AWS region (e.g., `us-east-2`)
- **Availability Zone** - AWS availability zone (e.g., `us-east-2a`)

This information is stored in a ConfigMap named `cluster-info-aws` in the `openshift-machine-api` namespace.

## Why This Exists

GPU MachineSets need cluster-specific values to work correctly. Without this automation, users would need to:
1. Manually find their cluster's availability zone
2. Edit parameter files or manifests
3. Commit changes to Git (problematic for shared repos)
4. Risk deploying GPU nodes in the wrong AZ (costly!)

With this component, everything is **auto-detected from your existing cluster infrastructure**. No manual edits, no Git commits required.

## Usage

### Via GitOps (Recommended)

```bash
# Deploy the detection job
oc apply -f gitops/infra/cluster-info-aws.yaml

# Wait for it to complete
oc wait --for=condition=complete job/generate-cluster-info-aws \
  -n openshift-machine-api --timeout=60s

# Verify the ConfigMap was created
oc get configmap cluster-info-aws -n openshift-machine-api -o yaml
```

### Manual Script (Alternative)

If you prefer not to use GitOps for this step:

```bash
chmod +x infra/cluster-info-aws/generate-cluster-info.sh
./infra/cluster-info-aws/generate-cluster-info.sh
```

## ConfigMap Output

The generated ConfigMap looks like this:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-info-aws
  namespace: openshift-machine-api
data:
  clusterName: "mycluster-abc123"
  infrastructureName: "mycluster-abc123-xyz456"
  platformType: "AWS"
  region: "us-east-2"
  availabilityZone: "us-east-2a"
```

## How It's Used

GPU MachineSet Kustomizations use [Kustomize replacements](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/replacements/) to inject these values:

```yaml
replacements:
  - source:
      kind: ConfigMap
      name: cluster-info-aws
      fieldPath: data.availabilityZone
    targets:
      - select:
          kind: MachineSet
        fieldPaths:
          - spec.template.spec.providerSpec.value.placement.availabilityZone
```

This means users can deploy GPU MachineSets directly from Git without any manual edits!

## Troubleshooting

**Job fails with "Could not determine cluster name"**:
- Ensure you have at least one existing MachineSet in your cluster
- Check: `oc get machineset -n openshift-machine-api`

**Job fails with "Could not determine availability zone"**:
- Ensure you have at least one existing Machine (worker node)
- Check: `oc get machines -n openshift-machine-api`

**ConfigMap exists but values are empty**:
- This is an AWS-only component. Verify your cluster is on AWS:
  ```bash
  oc get infrastructure cluster -o jsonpath='{.status.platformStatus.type}'
  ```

## How It Works

The Job runs a pod with the OpenShift CLI that:
1. Queries existing MachineSets to get the cluster name
2. Queries the Infrastructure resource to get region
3. Queries existing Machines to get the availability zone
4. Creates the ConfigMap with all detected values

The Job is configured with appropriate RBAC (ServiceAccount, ClusterRole, ClusterRoleBinding) to read cluster resources and create the ConfigMap.

After completion, the Job automatically cleans itself up after 5 minutes (`ttlSecondsAfterFinished: 300`).
