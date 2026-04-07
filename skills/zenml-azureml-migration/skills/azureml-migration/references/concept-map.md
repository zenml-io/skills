# AzureML SDK v2 -> ZenML Concept Map

Complete mapping of Azure Machine Learning SDK v2 concepts to their ZenML equivalents. Each entry is classified as **direct** (clean 1:1 mapping), **approximate** (conceptual equivalent with different semantics), or **absent** (no safe ZenML equivalent -- needs redesign or should stay Azure-native).

## Table of Contents

- [Orchestration Primitives](#orchestration-primitives)
- [Component Types](#component-types)
- [Environments](#environments)
- [Compute](#compute)
- [Data Passing and Parameters](#data-passing-and-parameters)
- [Control Flow and Scheduling](#control-flow-and-scheduling)
- [Model Management and Deployment](#model-management-and-deployment)
- [Azure Platform Features](#azure-platform-features)

## Orchestration Primitives

| AzureML Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `@pipeline` | `@pipeline`-decorated function | direct | Both define DAG-shaped workflows. ZenML expresses dependencies through step calls and artifact edges. |
| Pipeline parameters with defaults | Pipeline function parameters with Python defaults | direct | One of the cleanest translations. |
| Parameter override at submission time | Pipeline call / `.with_options(...)` / YAML config | direct | Same idea, different surface syntax. |
| `default_compute=` on pipeline | `AzureMLOrchestratorSettings(...)` on the pipeline | approximate | Similar intent, but compute moves from the Azure pipeline DSL into orchestrator settings. |
| `node.compute = ...` | Step-specific runtime configuration, step operator settings, or separate pipeline | approximate | Strongly heterogeneous per-node compute often needs deliberate redesign. |
| `CommandComponent` as the main executable unit | `@step` | direct | This is the primary migration path when authoring in ZenML. |
| Pipeline job submission | Running a ZenML pipeline on an AzureML-backed stack | approximate | Similar operational result, different authoring and control surfaces. |

## Component Types

| AzureML Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `@command_component` | `@step` | direct | Main translation path for Python-defined AzureML components. |
| YAML component + `load_component()` | Rewrite as a Python `@step` | approximate | ZenML has YAML config, but not YAML-defined pipeline components as first-class reusable assets. |
| Registered component | Python module/package, or keep Azure component external | absent | ZenML has no component registry equivalent. |
| Component versioning | Git/package version + pipeline snapshot/build metadata | absent | Similar governance goal, different primitive. |
| Pipeline component / sub-pipeline | Helper pipeline function or nested composition | approximate | Composition exists, but registered pipeline component assets do not. |
| Sweep component / `.sweep()` | Call HPO tooling or Azure sweep API inside a step | absent | No native ZenML sweep job primitive. |
| Parallel job / `parallel_run_function()` | Dynamic pipeline, `.map()`, or keep Azure job native | absent | Similar intent, not a safe 1:1 translation. |
| Spark component | Keep Spark execution external, or wrap submission in a step | approximate | No first-class ZenML Spark component abstraction. |
| AutoML job/component | Call Azure AutoML from a ZenML step, or replace with explicit training/HPO | approximate | Possible, but not native ZenML authoring. |
| Import/data transfer component | Wrap Azure SDK call in a step, or redesign ingestion | approximate | Usually depends on whether Azure-native asset management is required. |

## Environments

| AzureML Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `Environment(image=..., conda_file=...)` | `DockerSettings(parent_image=..., requirements=...)` | approximate | Same goal, different abstraction. ZenML treats this as container config, not a registered environment asset. |
| Curated environment | Prebuilt parent image | approximate | Similar operational outcome, but no curated environment catalog primitive in ZenML. |
| Custom environment via build context | `DockerSettings(dockerfile=..., build_context_root=...)` | approximate | Operationally close. |
| Environment asset registration/versioning | None | absent | Keep Azure environment assets if registry semantics matter. |
| Azure environment image caching | Build/image reuse | approximate | Similar effect, different lifecycle and visibility. |

## Compute

| AzureML Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Serverless compute | `mode="serverless"` | direct | Supported natively by the AzureML orchestrator. |
| Compute instance | `mode="compute-instance"` | direct | Includes instance name/size and idle shutdown options. |
| Compute cluster | `mode="compute-cluster"` | direct | Includes autoscaling and tier settings. |
| GPU VM size | `size="Standard_NC6s_v3"` (or similar) | direct | Preserved directly in orchestrator settings. |
| Dedicated / low-priority tier | `tier="Dedicated"` / `tier="LowPriority"` | direct | Good fit when keeping AzureML execution. |
| Autoscaling min/max | `min_instances`, `max_instances` | direct | Direct mapping in cluster mode. |
| Idle shutdown / scale-down | `idle_time_before_shutdown_minutes`, `idle_time_before_scaledown_down` | direct | Supported in compute instance / compute cluster modes respectively. |
| Managed network isolation | Azure workspace configuration | approximate | Preserved externally because execution still happens inside AzureML. |
| Attached Arc Kubernetes / Synapse Spark compute | None in AzureML orchestrator settings | absent | Not exposed as a first-class ZenML AzureML orchestrator mode. |

## Data Passing and Parameters

| AzureML Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Primitive inputs (`string`, `number`, `integer`, `boolean`) | Normal Python parameters | direct | Safe and straightforward. |
| Pipeline/component defaults | Python defaults | direct | Same intent. |
| `Input` / `Output` with asset descriptors | Step parameters and return values | approximate | Similar dataflow role, different type system. |
| `uri_file` | URI/path string or loaded Python object | approximate | Keep as a URI boundary when storage identity matters. |
| `uri_folder` | URI/path string or loaded collection/object | approximate | Same pattern as `uri_file`. |
| `mltable` | URI/reference or explicitly loaded DataFrame | approximate | No first-class MLTable type in ZenML. |
| `mlflow_model` | ZenML model artifact or path string | approximate | Similar purpose, different model-management surface. |
| `custom_model` / `triton_model` | Generic artifact or path | approximate | Usually handled as explicit artifacts or URIs. |
| Data assets | ZenML artifacts, or explicit Azure asset IDs passed through | approximate | Decide whether the asset identity itself is important. |
| Datastores | Artifact store / external storage configuration | approximate | Same storage concern, different abstraction boundary. |
| Data versioning | Artifact lineage/versioning, or keep Azure data assets external | approximate | Depends on whether Azure asset registry semantics are required. |
| Primitive outputs unsupported in AzureML | Primitive artifacts are normal in ZenML | direct | Behavioral difference that often simplifies the ZenML rewrite. |

## Control Flow and Scheduling

| AzureML Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Standard DAG dependency by passing outputs to inputs | Standard artifact dependency | direct | Core DAG pattern matches well. |
| Ordering without data transfer | Explicit dependency wiring / control dependency pattern | direct | ZenML can model ordering without pretending data exists. |
| `if_else` | Dynamic branching or redesign | absent | Treat as unsafe for automatic translation here. |
| `do_while` | Dynamic loop or keep loop inside one step | absent | Requires manual review and redesign. |
| `parallel_for` | Dynamic `.map()` / manual fan-out | absent | Similar shape, different guarantees and limitations. |
| `set_pipeline_controller_configurations` | None verified | absent | Treat as manual-review-only. |
| `JobSchedule` + cron trigger | `Schedule(cron_expression=...)` | direct | Works when using the AzureML orchestrator. |
| `JobSchedule` + recurrence trigger | `Schedule(start_time=..., interval_second=...)` | direct | Direct operational mapping. |
| Event triggers | None | absent | Not a native ZenML schedule primitive. |
| Update/delete Azure schedule through ZenML | Not native | absent | ZenML can create schedules, but lifecycle management remains manual on AzureML. |

### Orchestrator scheduling support

| Orchestrator | Scheduling support | Schedule types | Native schedule management |
|---|:---:|---|:---:|
| `AzureMLOrchestrator` | Yes | Cron, Interval | No |

## Model Management and Deployment

| AzureML Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Model registration/versioning | ZenML `Model` plus optional Azure registration step | approximate | ZenML models are first-class, but they are not Azure Registry entries unless you explicitly create those. |
| Managed online endpoint | No native equivalent | absent | Keep Azure-native, or call Azure SDK from a step. |
| Managed online deployment | No native equivalent | absent | Same reasoning. |
| Batch endpoint | No native equivalent | absent | Keep Azure-native if endpoint semantics matter. |
| MLflow tracking | MLflow inside ZenML step / experiment tracker | direct | Good fit when AzureML already exposes MLflow tracking. |
| MLflow model deployment to Azure endpoints | Azure SDK call from a ZenML step | approximate | Still possible, but it remains Azure deployment logic. |

## Azure Platform Features

| AzureML Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| AzureML Studio run UI | Still available when AzureML orchestrator is used | approximate | Preserved externally, not replaced by ZenML UI. |
| Designer | None | absent | ZenML is code-first, not visual pipeline authoring. |
| Responsible AI dashboard | None | absent | No equivalent dashboard workflow in ZenML. |
| AzureML Registry | None | absent | ZenML workspaces are not equivalent to Azure cross-workspace asset registry semantics. |
| Cross-workspace sharing of components/models/environments/data | None | absent | Keep Azure Registry if this is core to the operating model. |
| Managed network isolation | Azure workspace setting | approximate | Preserved because jobs still run on AzureML. |
