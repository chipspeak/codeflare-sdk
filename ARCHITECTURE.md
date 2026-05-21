# CodeFlare SDK -- Architectural Summary

## What is the CodeFlare SDK?

The CodeFlare SDK is a Python library that simplifies managing distributed computing resources on Kubernetes. It lets you create and manage [Ray](https://docs.ray.io/) clusters, submit batch jobs, and integrate with [Kueue](https://kueue.sigs.k8s.io/) for resource quota management -- all without writing Kubernetes YAML by hand (because Data Scientists wouldn't be fans!!).

It is maintained by Red Hat as part of the [project-codeflare](https://github.com/project-codeflare) organisation, licensed under Apache-2.0, and targets Python 3.11+.

### Key technologies the SDK wraps

| Technology | Role |
|---|---|
| **Ray** | A distributed computing framework. Ray clusters have a *head node* (coordinator, dashboard, GCS) and *worker nodes* (execute tasks). The SDK manages the lifecycle of these clusters on Kubernetes. |
| **KubeRay** | A Kubernetes operator that defines `RayCluster` and `RayJob` Custom Resources (CRs). The SDK generates these CRs from Python dataclasses and applies them via the Kubernetes API. |
| **Kueue** | A Kubernetes-native job queueing system. Workloads are submitted to a *LocalQueue* and admitted when cluster resources are available. The SDK labels CRs with queue metadata so Kueue can manage scheduling. |

---

## Repository layout

```
src/codeflare_sdk/
├── __init__.py                    # Public API surface
├── conftest.py                    # Global test fixtures (auto-mocks K8s clients)
│
├── common/                        # Shared infrastructure
│   ├── kubernetes_cluster/        # Auth, API client management, error handling
│   ├── kueue/                     # LocalQueue listing, default queue resolution
│   ├── utils/                     # Constants, helpers, TLS cert generation, validation
│   └── widgets/                   # Jupyter/IPython cluster management widgets
│
├── ray/
│   ├── cluster/                   # Interactive cluster lifecycle (Cluster, ClusterConfiguration)
│   ├── rayjobs/                   # Batch job submission (RayJob, ManagedClusterConfig)
│   └── client/                    # RayJobClient -- thin wrapper around Ray's JobSubmissionClient
│
└── vendored/                      # Vendored KubeRay Python client (DO NOT MODIFY)
```

Unit tests are colocated with source code (`src/codeflare_sdk/**/test_*.py`). E2E tests live in `tests/e2e/`.

---

## Architecture overview

The SDK has two main user-facing workflows, both of which ultimately create Kubernetes Custom Resources via the same shared infrastructure layer.

```
┌────────────────────────────────────────────────────┐
│                   User Code                        │
│  (Jupyter notebook, Python script, pipeline)       │
└──────────┬──────────────────────────┬──────────────┘
           │                          │
           ▼                          ▼
┌──────────────────────┐   ┌─────────────────────────┐
│  Interactive Path    │   │     Batch Path          │
│                      │   │                         │
│  ClusterConfiguration│   │  ManagedClusterConfig   │
│        ↓             │   │        ↓                │
│     Cluster          │   │     RayJob              │
│  .apply() / .down()  │   │  .submit() / .delete()  │
│  .status()           │   │  .status()              │
│  .wait_ready()       │   │  .stop() / .resubmit()  │
└─────────┬────────────┘   └──────────┬──────────────┘
          │                           │
          ▼                           ▼
┌─────────────────────────────────────────────────────────┐
│                 Shared Infrastructure                   │
│                                                         │
│  ┌──────────────┐ ┌────────────┐ ┌────────────────────┐ │
│  │ Auth /       │ │   Kueue    │ │   TLS / Certs      │ │
│  │ kube-authkit │ │ add_queue  │ │ generate_tls_cert  │ │
│  │ config_check │ │ list_local │ │  export_env        │ │
│  │ get_api_client││            │ │                    │ │
│  └──────────────┘ └────────────┘ └────────────────────┘ │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │     Kubernetes API (python kubernetes client)    │   │
│  │  CustomObjectsApi  ·  CoreV1Api  ·  DynamicClient│   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                     │
│                                                         │
│   KubeRay Operator  ·  Kueue Controller  ·  Ray Pods    │
└─────────────────────────────────────────────────────────┘
```

---

## Core components in detail

### 1. Authentication (`common/kubernetes_cluster/auth.py`)

All Kubernetes interaction begins with authentication. The SDK uses **[kube-authkit](https://github.com/opendatahub-io/kube-authkit)** as its primary authentication layer -- a mandatory dependency that provides unified Kubernetes authentication with auto-detection of credentials.

**Authentication priority in `config_check()`:**

1. **Existing global `api_client`** -- if already configured (e.g. via `set_api_client()`), reuse it.
2. **kube-authkit auto-detection** -- `AuthConfig(method="auto")` + `get_k8s_client()`. This handles kubeconfig files, in-cluster service accounts, OIDC, and other strategies transparently.
3. **Legacy fallback** -- `~/.kube/config` via `kubernetes.config.load_kube_config()`, or in-cluster config if `KUBERNETES_PORT` is set.

**Key functions:**

| Function | Role |
|---|---|
| `config_check()` | Ensures a valid K8s client exists; called before every API operation |
| `get_api_client()` | Returns the configured `kubernetes.client.ApiClient` singleton |
| `set_api_client(client)` | Injects a custom client (e.g. one created via kube-authkit directly) |

**kube-authkit re-exports:**

`AuthConfig` and `get_k8s_client` are re-exported at both `codeflare_sdk.common.kubernetes_cluster` and the top-level `codeflare_sdk` package, so users can authenticate directly:

```python
from codeflare_sdk import AuthConfig, get_k8s_client, set_api_client

auth = AuthConfig(method="kubeconfig")
client = get_k8s_client(config=auth)
set_api_client(client)
```

**Legacy classes** (`TokenAuthentication`, `KubeConfigFileAuthentication`) are deprecated and emit `DeprecationWarning` on instantiation. They still function but new code should use `AuthConfig` directly.

### 2. Interactive cluster lifecycle (`ray/cluster/`)

This is the primary workflow for data scientists working in jupyter notebooks.

**`ClusterConfiguration`** (dataclass) specifies the desired cluster shape: CPU/memory for head and workers, GPU requests, number of workers, autoscaling bounds, Kueue queue, volumes, tolerations, and more.

**`Cluster`** is the main entry point. Key methods:

| Method | What it does |
|---|---|
| `Cluster(config)` | Generates a `RayCluster` CR YAML from the configuration |
| `.apply()` | Server-side applies the CR to Kubernetes, then polls for TLS cert readiness |
| `.wait_ready()` | Blocks until the cluster is `READY` and the dashboard is accessible |
| `.status()` | Queries the `RayCluster` CR status and maps it to `CodeFlareClusterStatus` |
| `.details()` | Pretty-prints cluster resource information |
| `.down()` | Deletes the `RayCluster` CR and cleans up TLS certs |
| `.job_client` | Returns a `JobSubmissionClient` connected to the cluster dashboard |
| `.cluster_uri()` | Returns the `ray://` URI for `ray.init()` connections |
| `.cluster_dashboard_uri()` | Returns the dashboard URL (tries HTTPRoute, then OpenShift Route, then Ingress) |
| `.refresh_certificates()` | Regenerates TLS client certificates (useful after CA rotation) |

**`build_ray_cluster.py`** contains the logic that translates a `ClusterConfiguration` into a fully-formed `RayCluster` CR dict, including head/worker pod specs, resource limits, environment variables, and Kueue labels.

**Status enums** (`RayClusterStatus`, `CodeFlareClusterStatus`) model the cluster lifecycle. `RayClusterStatus` mirrors KubeRay's states (READY, UNHEALTHY, FAILED, SUSPENDED, UNKNOWN). `CodeFlareClusterStatus` adds SDK-specific states like STARTING and QUEUED.

### 3. Batch job submission (`ray/rayjobs/`)

For submitting self-contained jobs without manually managing cluster lifecycle.

**`ManagedClusterConfig`** (dataclass) is similar to `ClusterConfiguration` but tailored for RayJobs. It builds an embedded `rayClusterSpec` that KubeRay creates and tears down automatically.

**`RayJob`** manages the full job lifecycle:

| Method | What it does |
|---|---|
| `RayJob(...)` | Configures a job targeting either an existing cluster (`cluster_name`) or a new one (`cluster_config`) |
| `.submit()` | Builds the `RayJob` CR, handles local file packaging (zipping working directories into Kubernetes Secrets), and applies it |
| `.status()` | Queries the CR and maps `RayJobDeploymentStatus` to `CodeflareRayJobStatus` |
| `.stop()` / `.resubmit()` / `.delete()` | Lifecycle management via the vendored KubeRay Python client |

**Runtime environment handling** (`runtime_env.py`) automates packaging local files: single Python scripts are read and stored in a Kubernetes Secret, while directories are zipped and base64-encoded. An init container on the submitter pod unzips them before execution.

### 4. RayJobClient (`ray/client/ray_jobs.py`)

A thin wrapper around Ray's `JobSubmissionClient`. Used when you want to submit jobs directly to an already-running Ray cluster's dashboard (rather than creating a `RayJob` CR). Provides `submit_job()`, `list_jobs()`, `get_job_logs()`, `stop_job()`, etc.

### 5. Kueue integration (`common/kueue/kueue.py`)

Manages the interaction with Kueue's `LocalQueue` resources:

- **`list_local_queues()`** -- lists queues in a namespace, optionally filtering by resource flavors.
- **`get_default_kueue_name()`** -- finds the queue annotated with `kueue.x-k8s.io/default-queue: true`.
- **`add_queue_label()`** -- attaches the `kueue.x-k8s.io/queue-name` label to a CR so Kueue manages its scheduling.
- **`priority_class_exists()`** -- validates that a `WorkloadPriorityClass` exists before submission.

When a `Cluster` or `RayJob` specifies a `local_queue`, the SDK labels the CR accordingly. If no queue is specified, it auto-detects the default. If Kueue isn't installed, the label is simply omitted and the workload runs immediately.

### 6. TLS certificate management (`common/utils/generate_cert.py`)

Ray clusters on OpenShift use mTLS for secure communication. The SDK handles the full certificate lifecycle:

1. KubeRay creates a CA secret when the head pod starts.
2. `generate_tls_cert()` fetches the CA from Kubernetes and generates client certificates locally (signed by the cluster's CA).
3. `export_env()` sets environment variables (`RAY_USE_TLS`, `RAY_TLS_SERVER_CERT`, `RAY_TLS_SERVER_KEY`, `RAY_TLS_CA_CERT`) so `ray.init()` can connect securely.
4. `cleanup_tls_cert()` removes local certs when a cluster is torn down.
5. `refresh_tls_cert()` deletes and regenerates certificates (useful after CA rotation).

Certificates are stored under `~/.local/share/codeflare/tls/<cluster>-<namespace>/` by default. This can be overridden via the `CODEFLARE_TLS_DIR` environment variable or `XDG_DATA_HOME`.

### 7. Widgets (`common/widgets/widgets.py`)

Interactive IPython/Jupyter widgets for visual cluster management. `view_clusters()` renders a table of all clusters in a namespace with buttons to apply, delete, or inspect them. These widgets are automatically displayed when a `Cluster` object is created inside a notebook.

### 8. Vendored KubeRay client (`vendored/`)

A vendored copy of the KubeRay Python client. Provides `RayjobApi` and `RayClusterApi` for direct CR manipulation. This directory should never be modified. The `RayJob` class uses the vendored client for job submission and lifecycle operations; the interactive `Cluster` class uses the standard `kubernetes` Python client (`CustomObjectsApi`, `DynamicClient`) directly.

---

## Public API surface (`__init__.py`)

Everything exported from the top-level `codeflare_sdk` package:

| Export | Source | Purpose |
|---|---|---|
| `Cluster`, `ClusterConfiguration` | `ray.cluster` | Interactive cluster lifecycle |
| `RayClusterStatus`, `CodeFlareClusterStatus`, `RayCluster` | `ray.cluster.status` | Cluster status enums and dataclass |
| `get_cluster` | `ray.cluster.cluster` | Retrieve an existing cluster as a `Cluster` object |
| `list_all_clusters` | `ray.cluster.cluster` | List all `RayCluster` CRs in a namespace |
| `list_all_queued` | `ray.cluster.cluster` | List clusters waiting for Kueue admission |
| `RayJob`, `ManagedClusterConfig` | `ray.rayjobs` | Batch job submission |
| `RayJobClient` | `ray.client` | Direct Ray dashboard job submission wrapper |
| `view_clusters` | `common.widgets` | Jupyter widget for cluster management |
| `AuthConfig`, `get_k8s_client` | `kube-authkit` (re-exported) | Authentication |
| `set_api_client` | `common.kubernetes_cluster` | Inject a custom K8s API client |
| `Authentication`, `TokenAuthentication`, `KubeConfigFileAuthentication`, `KubeConfiguration` | `common.kubernetes_cluster` | Legacy auth (deprecated) |
| `list_local_queues` | `common.kueue` | List Kueue LocalQueues |
| `generate_cert` | `common.utils` | TLS certificate generation module |
| `copy_demo_nbs` | `common.utils.demos` | Copy demo notebooks to working directory |

---

## Data flow: creating and using a cluster

```python
from codeflare_sdk import Cluster, ClusterConfiguration

# 1. Configure
config = ClusterConfiguration(
    name="my-cluster",
    namespace="my-ns",
    num_workers=2,
    worker_cpu_requests=2,
    worker_memory_requests=8,
    worker_extended_resource_requests={"nvidia.com/gpu": 1},
    local_queue="user-queue",
)

# 2. Create and apply
cluster = Cluster(config)     # Generates RayCluster CR YAML
cluster.apply()               # Applies CR + generates TLS certs
cluster.wait_ready()          # Polls until READY + dashboard up

# 3. Use
ray.init(address=cluster.cluster_uri())
# ... run distributed work ...

# 4. Tear down
cluster.down()                # Deletes CR + cleans up certs
```

Under the hood, step 2 involves: `config_check()` -> `build_ray_cluster()` (CR generation with Kueue labels) -> `DynamicClient.server_side_apply()` -> poll for CA secret -> `generate_tls_cert()`.

---

## Data flow: submitting a batch RayJob

```python
from codeflare_sdk import RayJob, ManagedClusterConfig

job = RayJob(
    job_name="training-job",
    entrypoint="python train.py",
    cluster_config=ManagedClusterConfig(num_workers=4),
    runtime_env={"working_dir": "./my-scripts", "pip": ["torch"]},
    local_queue="gpu-queue",
)

job.submit()                   # Creates RayJob CR (cluster auto-managed)
status, done = job.status()    # Polls job deployment status
job.delete()                   # Cleans up CR and associated cluster
```

Here the SDK zips `./my-scripts/` into a Kubernetes Secret, builds a `submitterPodTemplate` with an init container to unzip it, embeds a `rayClusterSpec` for the auto-managed cluster, labels the CR for Kueue, and applies everything via the vendored KubeRay client.

---

## Error handling patterns

The codebase uses a consistent pattern for Kubernetes API errors:

1. Call `config_check()` to ensure authentication is configured.
2. Use `get_api_client()` to obtain the client.
3. Wrap API calls in `try/except ApiException`.
4. Route exceptions through `_kube_api_error_handling(e)` (defined in `common/kubernetes_cluster/kube_api_helpers.py`).

Custom Resource dicts are parsed defensively with `.get()` and isolated `try/except` blocks for each section (metadata, spec, status). Missing or invalid status values map to `UNKNOWN` enum variants.

---

## Testing architecture

- **Global fixtures** (`conftest.py`): Auto-mocks `kubernetes.client.AuthenticationApi`, `CoreV1Api`, and `CustomObjectsApi` so no real Kubernetes calls are made. Creates a minimal kubeconfig and TLS fixtures.
- **Test helpers** (`common/utils/unit_test_support.py`): Functions like `get_ray_obj_with_status()`, `create_cluster_config()`, and `apply_template()` generate test fixtures. Raw Kubernetes JSON payloads should never be hardcoded in tests.
- **Coverage**: Project target is 90% overall (CI), 85% per patch (new/changed code).

---

## Key dependencies

| Package | Purpose |
|---|---|
| `ray[data,default]` 2.54.1 | Ray framework + data processing |
| `kubernetes` >= 27.2 | Kubernetes Python client |
| `kube-authkit` >= 0.4 | Primary authentication layer -- auto-detects kubeconfig, in-cluster, OIDC, and other K8s auth strategies. `AuthConfig` and `get_k8s_client` are re-exported at the SDK top level. |
| `cryptography` 46.x | TLS certificate generation |
| `ipywidgets` 8.x | Jupyter interactive widgets |
| `rich` >= 12.5 | Terminal pretty-printing |
| `pydantic` >= 2.10 | Data validation |
| `openshift-client` 1.0.18 | OpenShift CLI integration |
