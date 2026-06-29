# DynamoGraphDeployment (DGD) Spec Reference

## `spec`

**Group:** `nvidia.com`  
**Kind:** `DynamoGraphDeployment`  
**Version:** `v1beta1`

### Description

Defines the desired state for a Dynamo graph deployment.

### Fields

| Field | Type | Description |
|---------|------|-------------|
| `annotations` | `map[string]string` | Annotations propagated to child resources. Component-level values take precedence. |
| `backendFramework` | `string` | Backend framework (`sglang`, `vllm`, `trtllm`). |
| `components` | `[]object` | Components deployed as part of the graph. |
| `env` | `[]object` | Environment variables prepended to every component. |
| `labels` | `map[string]string` | Labels propagated to child resources. |
| `restart` | `object` | Graph deployment restart policy. |
| `topologyConstraint` | `object` | Deployment-level topology constraints. |

---

# `spec.components`

### Description

Defines the components deployed as part of the graph.

Each component:

- Has a unique logical `name`
- Represents a deployable Dynamo workload
- Can be repeated except for `type: epp`, which may appear at most once

### v1beta1 Changes

The ten component pod-configuration fields from v1alpha1:

- resources
- envs
- envFromSecret
- livenessProbe
- readinessProbe
- volumeMounts
- annotations
- labels
- extraPodMetadata
- extraPodSpec

have been replaced with a single:

```text
podTemplate
```

which uses a native Kubernetes `PodTemplateSpec`.

### Fields

| Field | Type | Description |
|---------|------|-------------|
| `compilationCache` | `object` | PVC-backed compilation cache configuration. |
| `eppConfig` | `object` | Endpoint Picker Plugin configuration. |
| `experimental` | `object` | Experimental features not covered by normal API guarantees. |
| `frontendSidecar` | `string` | Container name designated as the frontend sidecar. |
| `globalDynamoNamespace` | `boolean` | Deploy component in the global Dynamo namespace. |
| `modelRef` | `object` | Model served by the component. |
| `multinode` | `object` | Multinode deployment configuration. |
| `name` | `string` **required** | Stable logical component identifier. |
| `podTemplate` | `object` | Kubernetes pod template used for component pods. |
| `replicas` | `integer` | Desired replica count. |
| `scalingAdapter` | `object` | Enables DGDSA-managed scaling. |
| `sharedMemorySize` | `object` | Size of `/dev/shm` tmpfs volume. |
| `topologyConstraint` | `object` | Component-level topology constraints. |
| `type` | `string` | Component role within the graph. |

---

# `spec.components.compilationCache`

### Description

Configures a PVC-backed compilation cache.

The operator automatically:

- Mounts the cache
- Configures backend-specific paths
- Injects required environment variables

Users do not need to manually configure these through `podTemplate`.

### Fields

### `pvcName`

**Type:** `string` (required)

References a user-created PVC.

Requirements:

- Must exist in the same namespace as the DGD.

---

### `mountPath`

**Type:** `string`

Overrides the backend-specific default mount path.

When omitted:

- The operator selects a backend-specific default.

---

# `spec.components.eppConfig`

### Description

Configuration for Endpoint Picker Plugin (EPP) components.

Only meaningful when:

```text
type = epp
```

### Fields

### `config`

**Type:** `object`

Provides EPP configuration directly as a structured object.

Behavior:

- Marshaled to YAML automatically
- Stored in a generated ConfigMap

---

### `configMapRef`

**Type:** `object`

References a user-managed ConfigMap containing EPP configuration.

### Notes

- `config` and `configMapRef` are mutually exclusive.
- One must be specified.

---

# `spec.components.experimental`

### Description

Contains preview features that may change or be removed in future releases.

These fields are **not covered** by normal v1beta1 compatibility guarantees.

Not recommended for production workloads.

### Fields

### `checkpoint`

Container-image snapshotting and restore support.

Enables:

- DynamoCheckpoint creation from running pods
- Pod restoration from checkpoints
- Faster cold starts

---

### `failover`

Active-passive GPU failover configuration.

Requirements:

- `gpuMemoryService` must also be enabled.
- Modes must match.

Validation is enforced by the webhook.

---

### `gpuMemoryService`

Configures the GPU Memory Service (GMS) sidecar.

When enabled:

- GMS sidecar is injected
- GPU access is managed through DRA

---

# `spec.components.modelRef`

### Description

References a model served by this component.

When specified:

- A headless service is created for endpoint discovery.

### Fields

### `name`

**Type:** `string` (required)

Base model identifier.

Example:

```text
llama-3-70b-instruct-v1
```

---

### `revision`

**Type:** `string`

Model revision or version.

---

# `spec.components.multinode`

### Description

Configuration for multinode deployments.

### Fields

### `nodeCount`

**Type:** `integer`

Number of nodes used by the component.

Total GPU consumption:

```text
nodeCount × GPU request per container
```

---

# `spec.components.podTemplate`

### Description

Native Kubernetes `PodTemplateSpec`.

The operator injects defaults into the container named:

```text
main
```

Injected defaults include:

- image
- command
- environment variables
- ports
- probes
- resources
- volume mounts

### Behavior

If a container named `main` exists:

- User configuration is merged by name.

If `main` does not exist:

- The operator creates it automatically.

All other containers are treated as sidecars and are fully user-managed.

### Fields

| Field | Type |
|---------|------|
| `metadata` | `object` |
| `spec` | `object` |

---

# `spec.components.type`

### Description

Defines the role of the component within the graph.

Used for:

- Port mapping
- Frontend detection
- Planner RBAC
- Component labeling

### Valid Values

| Value | Description |
|---------|-------------|
| `frontend` | Request ingress/frontend component. |
| `worker` | General worker component. |
| `prefill` | Prefill stage component. |
| `decode` | Decode stage component. |
| `planner` | Planner component. |
| `epp` | Endpoint Picker Plugin component. |

---

# `spec.components.frontendSidecar`

### Description

Identifies a container within `podTemplate.spec.containers` that should act as the frontend sidecar.

Requirements:

- Value must match an existing container name.
- Validation webhook rejects invalid names.

The operator merges frontend-sidecar defaults into the designated container.

Injected defaults include:

- Dynamo environment variables
- Ports
- Health probes

---

# `spec.components.scalingAdapter`

### Description

Opts the component into DynamoGraphDeploymentScalingAdapter (DGDSA) management.

When enabled:

- DGDSA owns the `replicas` field.
- External autoscalers can scale through the Scale subresource.
- Manual modification of `replicas` is discouraged.

---

# `spec.components.sharedMemorySize`

### Description

Controls the tmpfs volume mounted at:

```text
/dev/shm
```

### Behavior

| Value | Result |
|---------|---------|
| `nil` | Operator default (8Gi). |
| Positive quantity | Custom size. |
| `"0"` | Disable shared-memory volume. |

---

# `spec.components.globalDynamoNamespace`

### Description

Deploys the component into the global Dynamo namespace instead of the deployment-specific namespace.

---

# `spec.topologyConstraint`

### Description

Deployment-level topology constraint.

When specified:

- `clusterTopologyName` identifies the ClusterTopology resource.
- `packDomain` is optional.

Components without their own topology constraints inherit this value.

---

# `spec.env`

### Description

Environment variables prepended to every component.

Component-level variables with the same name take precedence.

---

# `spec.annotations`

### Description

Annotations propagated to:

- PCS resources
- DCD resources
- Deployments
- Pod templates

Component-level annotations override deployment-level values.

---

# `spec.labels`

### Description

Labels propagated to all child resources.

Component-level labels override deployment-level values.


### Example YAML
```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeployment
metadata:
  name: qwen3-direct
  namespace: default
  # annotations:                          
  #   nvidia.com/enable-grove: "false"  ## Forcefully disables grove for deployment
spec:
  annotations:
  backendFramework: vllm # or "trtllm" or "sglang"
  components:
    - compilationCache:
        mountPath:
        pvcName:
      eppConfig:
        config:
        configMapRef:
      experimental:
        checkpoint:
        failover:
        gpuMemoryService:
      frontendSidecar:
      globalDynamoNamespace:
      modelRef:
        name:
        revision:
      multinode:
        nodeCount: 
      name:
      podTemplate:
        metadata:
          name:
          namespace:
        spec:
          containers:
          imagePullSecrets:
          initContainers:
          nodeName:
          nodeSelector:
          resources:
          resourceClaims:
          volumes:
      replicas:
      type: "frontend" # or "worker", "prefill", "decode", "planner", "epp"
      enum:
  env: 
  labels:
  restart:
  topologyConstraint:
```
