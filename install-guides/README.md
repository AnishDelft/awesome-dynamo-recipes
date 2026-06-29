# Install Guide: Dynamo Operator

This guide documents the steps to install the NVIDIA Dynamo Operator on a Kubernetes cluster in three supported configurations:

- Dynamo Operator
- Dynamo Operator with Grove
- Dynamo Operator with Grove and KAI Scheduler

The Dynamo Operator is installed as part of the `dynamo-platform` Helm chart. The platform chart also installs supporting services used by Dynamo, including NATS by default. etcd can be enabled when your deployment requires it.

> Note: Grove is used for multinode and disaggregated Dynamo deployments. Grove requires a compatible gang scheduler. If KAI Scheduler is not installed by this guide, confirm that another supported scheduler is already installed and configured.

> Note: This is tested on `v1.2.0`.
## Table of Contents

- [Prerequisites](#prerequisites)
- [Set Installation Variables](#set-installation-variables)
- [Install the NVIDIA GPU Operator](#install-the-nvidia-gpu-operator)
- [Option 1: Install Dynamo Operator](#option-1-install-dynamo-operator)
- [Option 2: Install Dynamo Operator with Grove](#option-2-install-dynamo-operator-with-grove)
- [Option 3: Install Dynamo Operator with Grove and KAI Scheduler](#option-3-install-dynamo-operator-with-grove-and-kai-scheduler)
- [Verify the Installation](#verify-the-installation)
- [Upgrade an Existing Installation](#upgrade-an-existing-installation)
- [Uninstall](#uninstall)
- [Troubleshooting](#troubleshooting)
- [References](#references)

## Prerequisites

Before you begin, confirm the following:

- Kubernetes cluster version `v1.24` or later.
- GPU-capable worker nodes are available.
- `kubectl` is installed and configured for the target cluster.
- Helm `v3.0` or later is installed.
- Cluster-admin permissions are available for installing cluster-scoped CRDs, roles, and webhooks.

Verify local tooling:

```bash
kubectl version --client
helm version
kubectl config current-context
kubectl get nodes -o wide
```

If the cluster already has a cluster-wide Dynamo Operator, do not install a second one:

```bash
kubectl get clusterrolebinding -o name | grep dynamo-operator-manager
```

If the command returns an existing binding, identify the owner namespace and coordinate with the cluster administrator before continuing.

## Set Installation Variables

Set the namespace and release versions used by the commands in this guide:

```bash
export DYNAMO_NAMESPACE=dynamo-system
export DYNAMO_VERSION=1.2.1
export DYNAMO_CHART="https://helm.ngc.nvidia.com/nvidia/ai-dynamo/charts/dynamo-platform-${DYNAMO_VERSION}.tgz"

# Required only when installing Grove or KAI Scheduler separately.
# Use versions from the upstream release pages that are compatible with your Dynamo release.
export GROVE_NAMESPACE=grove-system
export GROVE_VERSION=v0.1.0-alpha.9

export KAI_NAMESPACE=kai-scheduler
export KAI_VERSION=v0.16.0
```

For production environments, pin all versions explicitly and track them in the repository.

## Install the NVIDIA GPU Operator

The GPU Operator installs and manages the NVIDIA driver, container toolkit, device plugin, and GPU monitoring components.

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

helm upgrade --install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace
```

If GPU drivers are already installed by the cloud provider or base image, disable GPU Operator driver management:

```bash
helm upgrade --install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace \
  --set driver.enabled=false
```

Verify the GPU Operator:

```bash
kubectl get pods -n gpu-operator
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPUS:.status.allocatable.'nvidia\.com/gpu'
```

## Option 1: Install Dynamo Operator

Use this option for a base Dynamo Operator installation without Grove or KAI Scheduler.

```bash
helm upgrade --install dynamo-platform \
  "${DYNAMO_CHART}" \
  --namespace "${DYNAMO_NAMESPACE}" \
  --create-namespace
```

This installs the Dynamo Platform services and the Dynamo Operator in cluster-wide mode.

## Option 2: Install Dynamo Operator with Grove

Use this option when Grove is required, but KAI Scheduler is installed separately or another supported gang scheduler is already available.

### Option 2A: Install Grove as a bundled Dynamo subchart

```bash
helm upgrade --install dynamo-platform \
  "${DYNAMO_CHART}" \
  --namespace "${DYNAMO_NAMESPACE}" \
  --create-namespace \
  --set global.grove.install=true
```

### Option 2B: Install Grove externally and enable it in Dynamo

Install Grove:

```bash
helm upgrade --install grove \
  oci://ghcr.io/ai-dynamo/grove/grove-charts \
  --version "${GROVE_VERSION}" \
  --namespace "${GROVE_NAMESPACE}" \
  --create-namespace
```

Install or update Dynamo Platform with Grove enabled:

```bash
helm upgrade --install dynamo-platform \
  "${DYNAMO_CHART}" \
  --namespace "${DYNAMO_NAMESPACE}" \
  --create-namespace \
  --set global.grove.enabled=true
```

Verify Grove resources:

```bash
kubectl get crd | grep -E 'grove.io|scheduler.grove.io'
kubectl get pods -n "${GROVE_NAMESPACE}"
```

## Option 3: Install Dynamo Operator with Grove and KAI Scheduler

Use this option for multinode or disaggregated Dynamo deployments where Grove and KAI Scheduler should be installed together.

### Option 3A: Install Grove and KAI Scheduler as bundled Dynamo subcharts

This path is the simplest for development and test clusters:

```bash
helm upgrade --install dynamo-platform \
  "${DYNAMO_CHART}" \
  --namespace "${DYNAMO_NAMESPACE}" \
  --create-namespace \
  --set global.grove.install=true \
  --set global.kai-scheduler.install=true
```

### Option 3B: Install KAI Scheduler and Grove externally, then enable them in Dynamo

This path is recommended when platform components are managed independently or shared across multiple namespaces.

Install KAI Scheduler:

```bash
helm upgrade --install kai-scheduler \
  oci://ghcr.io/kai-scheduler/kai-scheduler/kai-scheduler \
  --version "${KAI_VERSION}" \
  --namespace "${KAI_NAMESPACE}" \
  --create-namespace
```

Do not submit workloads into the `kai-scheduler` namespace. Use a dedicated workload namespace.

Install Grove:

```bash
helm upgrade --install grove \
  oci://ghcr.io/ai-dynamo/grove/grove-charts \
  --version "${GROVE_VERSION}" \
  --namespace "${GROVE_NAMESPACE}" \
  --create-namespace
```

Install or update Dynamo Platform with both integrations enabled:

```bash
helm upgrade --install dynamo-platform \
  "${DYNAMO_CHART}" \
  --namespace "${DYNAMO_NAMESPACE}" \
  --create-namespace \
  --set global.grove.enabled=true \
  --set global.kai-scheduler.enabled=true
```

Verify KAI Scheduler and Grove:

```bash
kubectl get pods -n "${KAI_NAMESPACE}"
kubectl get pods -n "${GROVE_NAMESPACE}"
kubectl get crd | grep -E 'podgroups|grove.io|scheduler.grove.io'
```

## Verify the Installation

Check Dynamo CRDs:

```bash
kubectl get crd | grep dynamo
```

Expected Dynamo CRDs include resources such as:

- `dynamographdeployments.nvidia.com`
- `dynamocomponentdeployments.nvidia.com`
- `dynamographdeploymentrequests.nvidia.com`

Check platform pods:

```bash
kubectl get pods -n "${DYNAMO_NAMESPACE}"
```

Expected pods include:

- `dynamo-operator-*`
- `nats-*`

If etcd is enabled with `--set global.etcd.install=true`, expect `etcd-*` pods as well.

Check operator logs:

```bash
kubectl logs -n "${DYNAMO_NAMESPACE}" deploy/dynamo-operator-controller-manager --all-containers=true
```

If you are running this from the Dynamo source repository, run the pre-deployment check:

```bash
./deploy/pre-deployment/pre-deployment-check.sh
```

## Upgrade an Existing Installation

Upgrade Dynamo Platform while preserving existing values:

```bash
helm upgrade dynamo-platform \
  "${DYNAMO_CHART}" \
  --namespace "${DYNAMO_NAMESPACE}" \
  --reuse-values
```

Enable Grove and KAI Scheduler on an existing installation:

```bash
helm upgrade dynamo-platform \
  "${DYNAMO_CHART}" \
  --namespace "${DYNAMO_NAMESPACE}" \
  --reuse-values \
  --set global.grove.enabled=true \
  --set global.kai-scheduler.enabled=true
```

## Uninstall

Uninstall Dynamo Platform:

```bash
helm uninstall dynamo-platform --namespace "${DYNAMO_NAMESPACE}"
```

If Grove was installed externally:

```bash
helm uninstall grove --namespace "${GROVE_NAMESPACE}"
```

If KAI Scheduler was installed externally:

```bash
helm uninstall kai-scheduler --namespace "${KAI_NAMESPACE}"
```

CRDs are cluster-scoped and may remain after Helm uninstall. Review carefully before deleting CRDs because deleting CRDs also deletes their custom resources:

```bash
kubectl get crd | grep -E 'dynamo|grove|podgroup'
```

Delete CRDs only when the cluster no longer has workloads depending on them.

## Troubleshooting

### Dynamo CRDs are missing

Check the Helm release:

```bash
helm status dynamo-platform --namespace "${DYNAMO_NAMESPACE}"
kubectl get events -n "${DYNAMO_NAMESPACE}" --sort-by=.lastTimestamp
```

### Dynamo Operator pod is not running

Inspect the pod and logs:

```bash
kubectl get pods -n "${DYNAMO_NAMESPACE}"
kubectl describe pod -n "${DYNAMO_NAMESPACE}" <pod-name>
kubectl logs -n "${DYNAMO_NAMESPACE}" <pod-name> --all-containers=true
```

### Multinode workloads fail without Grove or LWS

Multinode Dynamo deployments require Grove or an alternative orchestrator such as LeaderWorkerSet with Volcano. Install Grove and configure a supported scheduler before retrying the workload.

### Grove workloads remain pending

Confirm the scheduler is installed and the cluster has enough resources to satisfy gang scheduling:

```bash
kubectl get pods -A | grep -E 'kai|grove'
kubectl describe nodes
kubectl get events -A --sort-by=.lastTimestamp
```

### KAI Scheduler is installed but workloads are not using it

Confirm the workload namespace and workload manifests are configured to use the expected scheduler class or scheduling integration. Do not submit workloads into the `kai-scheduler` namespace.

### Bitnami etcd image is rejected

If the platform install fails because the Bitnami etcd image is treated as unrecognized, add the following values to the Dynamo Platform Helm command:

```bash
--set "etcd.image.repository=bitnamilegacy/etcd" \
--set "etcd.global.security.allowInsecureImages=true"
```

## References

- [NVIDIA Dynamo Kubernetes Installation Guide](https://docs.nvidia.com/dynamo/dev/kubernetes-deployment/start-here/installation-guide)
- [NVIDIA Dynamo Operator Documentation](https://docs.nvidia.com/dynamo/dev/kubernetes-deployment/start-here/dynamo-operator)
- [NVIDIA Dynamo Release Artifacts](https://docs.nvidia.com/dynamo/dev/resources/release-artifacts)
- [Grove Installation Guide](https://github.com/ai-dynamo/grove/blob/main/docs/installation.md)
- [KAI Scheduler README](https://github.com/kai-scheduler/KAI-Scheduler)


