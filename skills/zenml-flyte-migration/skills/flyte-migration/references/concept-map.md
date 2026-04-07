# Flyte → ZenML Concept Map

Complete mapping of Flyte concepts to their ZenML equivalents. Each entry is classified as **direct** (clean 1:1 mapping), **approximate** (conceptual equivalent with different semantics), **absent** (no safe ZenML equivalent), or **absent / unknown** where the public Union/Flyte material was too thin for a confident migration claim.

## Table of Contents

- [Core Workflow Primitives](#core-workflow-primitives)
- [Type System and Data Passing](#type-system-and-data-passing)
- [Execution Features](#execution-features)
- [Infrastructure and Containerization](#infrastructure-and-containerization)
- [Scheduling, LaunchPlans, and Notifications](#scheduling-launchplans-and-notifications)
- [Plugin-backed and External Execution Patterns](#plugin-backed-and-external-execution-patterns)
- [Union / Commercial Extensions](#union--commercial-extensions)
- [Orchestrator Scheduling Support](#orchestrator-scheduling-support)

## Core Workflow Primitives

| Flyte Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `@task` | `@step` | approximate | Closest basic execution unit, but Flyte task interfaces and runtime semantics are richer. |
| `@workflow` | `@pipeline` | approximate | Both compose execution units, but registration and trigger semantics differ. |
| nested workflow / subworkflow | nested pipeline composition or wrapper pipeline | approximate | Works conceptually, but registration and LaunchPlan behavior do not carry over 1:1. |
| `@dynamic` | `@pipeline(dynamic=True)` | approximate | Both allow runtime-shaped DAGs, but ZenML uses `.load()` and has orchestrator limits. |
| `@eager` | no direct core equivalent | absent | Treat as a redesign boundary, not a decorator swap. |
| Flyte promises / node outputs | step invocation handles + artifacts | approximate | Similar authoring feel, different engine semantics. |
| `.with_overrides(...)` | `.with_options(...)` + settings | approximate | Similar intent, different scope and backend support. |

## Type System and Data Passing

| Flyte Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| primitives and collections | same Python types | direct | Best-case migration. |
| `Enum` | `str`, `Literal[...]`, or enum + custom materializer | approximate | Preserve the closed value set explicitly. |
| dataclass values | dataclass + materializer / JSON model | approximate | Make serialization explicit. |
| Pydantic `BaseModel` | dict / dataclass / custom materializer | approximate | Preserve validation intentionally. |
| `Annotated[T, metadata]` | `Annotated[T, "artifact_name"]` + explicit metadata logging | absent / approximate | Flyte metadata-bearing annotations do not map cleanly. |
| `FlyteFile` | `Path` artifact | approximate | Recreate remote-reference / provenance semantics when needed. |
| `FlyteDirectory` | `Path` to directory artifact | approximate | Same warning as `FlyteFile`. |
| `FlyteSchema` | dataframe or table artifact + validator | approximate | Prefer explicit schema checks. |
| `StructuredDataset` | `pd.DataFrame`, `polars.DataFrame`, or `pyarrow.Table` artifact | approximate | Recreate storage / format metadata intentionally. |
| arbitrary Python object | custom materializer or temporary `CloudpickleMaterializer` | absent / approximate | Cloudpickle is a transition tactic, not the desired endpoint. |

## Execution Features

| Flyte Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `map_task()` | dynamic pipeline `.map()` | approximate | Similar purpose, different runtime and backend semantics. |
| `map_task(concurrency=...)` | orchestrator-dependent parallelism controls | absent / approximate | No portable 1:1 core knob. |
| `map_task(min_success_ratio=...)` | custom reducer / error-tolerant redesign | absent | Must be redesigned explicitly. |
| `conditional()` | `@pipeline(dynamic=True)` + `.load()` + Python control flow | approximate | Similar branching goal, different execution semantics. |
| `cache=True` | `@step(enable_cache=True)` | approximate | Similar intent. |
| `cache_version="..."` | explicit cache-buster parameter or config | approximate | No first-class identical ZenML core field is documented in the existing repo guidance. |
| retries | `StepRetryConfig(max_retries=...)` | approximate | Usually a good translation. |
| timeout | backend-specific timeout handling | absent / approximate | Treat as target-stack-specific unless verified. |
| `interruptible=True` | backend-level spot / preemptible config | absent | No portable ZenML core equivalent. |
| intra-task checkpointing | split into smaller steps + persisted artifacts | absent | Strong redesign boundary. |
| Decks | metadata logging, report artifacts, visualizers | approximate | No direct Deck object equivalent. |

## Infrastructure and Containerization

| Flyte Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `Resources(cpu=..., mem=..., gpu=...)` | `ResourceSettings(cpu_count=..., memory=..., gpu_count=...)` | approximate | Same intent, different enforcement model. |
| requests / limits split | `ResourceSettings(...)` | approximate | Flyte is more explicit about requests vs limits. |
| `ImageSpec(...)` | `DockerSettings(...)` | approximate | Similar purpose, different build lifecycle. |
| per-task image overrides | step-level Docker settings | approximate | Possible, but not identical to Flyte image packaging. |
| Flyte secrets access | ZenML secrets store / service connectors | approximate | Similar outcome, different abstraction. |
| `ContainerTask` | wrapper step / step operator / external job submission | absent | No first-class raw container task primitive in ZenML core. |

## Scheduling, LaunchPlans, and Notifications

| Flyte Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `LaunchPlan.get_or_create()` | deployment / `with_options(schedule=...)` + config | approximate | Schedule maps; registered execution surface does not. |
| default LaunchPlan | default pipeline invocation / deployment config | approximate | Not a first-class registered object in the same way. |
| `default_inputs` | pipeline defaults / config | approximate | Usually straightforward. |
| `fixed_inputs` | wrapper pipeline or deployment-specific hard-coded config | approximate | Must be made explicit. |
| scheduled LaunchPlan | `Schedule(...)` | approximate | Only the scheduling slice maps directly. |
| LaunchPlan notifications | step hooks / alerter hooks / external alerts | approximate | Different abstraction level. |
| embedded LaunchPlans / separate execution | wrapper pipeline or external orchestration | absent / approximate | Do not assume nested execution identity matches. |
| `reference_workflow` | no direct equivalent | absent | Redesign as a shared library boundary, API/service boundary, or explicit artifact exchange. |
| `reference_task` | no direct equivalent | absent | Cross-project registered entity references do not map cleanly to ZenML core. |
| `reference_launch_plan` | no direct equivalent | absent | Cross-project registered references must be redesigned. |

## Plugin-backed and External Execution Patterns

| Flyte Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Spark plugin | Spark step operator / external Spark job step | approximate | Good intent-level match. |
| Ray plugin | Ray step operator / external Ray launcher | approximate | Similar target, different integration abstraction. |
| Dask plugin | Dask step operator / external Dask submission | approximate | Same warning. |
| MPI / Kubeflow MPI plugin | custom operator or external launcher | absent / approximate | No first-class core equivalent. |
| SageMaker plugin | SageMaker orchestrator / step operator / service integration | approximate | Depends on the target ZenML stack. |
| BigQuery / Snowflake / SQL family | normal SDK / SQL client step | approximate | Usually rewrite as explicit code or external job boundary. |
| dbt plugin | dbt step / containerized dbt run / external service | approximate | Usually a wrapper step. |
| Databricks plugin / tasks | Databricks step operator or external job submission | approximate | Translate the boundary, not the config object. |
| notebook / papermill-style external execution | wrapper step or external service | approximate | Keep inputs and outputs explicit. |

## Union / Commercial Extensions

| Union Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Union Cloud managed Flyte hosting | ZenML server / managed platform | approximate | Same broad problem space, different platform semantics. |
| Union RBAC | ZenML Pro / server-side multi-user controls | approximate | Do not assume a 1:1 policy model. |
| control plane / data plane split | platform architecture choice | approximate | Architectural similarity, not a code-level mapping. |
| reusable containers / `ReusePolicy` | long-lived runners / service-style optimization | absent / approximate | No direct ZenML core equivalent. |
| Union Actors / persistent execution | no direct core equivalent | absent | Strong redesign boundary. |
| Union Artifacts | unclear stable public equivalent | absent / unknown | Preserve the uncertainty; do not over-claim a mapping. |
| Union Channels | unclear stable public equivalent | absent / unknown | Public semantics were not clear enough for a safe migration claim. |

## Orchestrator Scheduling Support

ZenML scheduling support is orchestrator-dependent. This matters because Flyte LaunchPlan scheduling is a platform primitive, while ZenML `Schedule` only works where the active orchestrator supports it.

| ZenML Orchestrator | Scheduling Support | Supported Types |
|---|---|---|
| Airflow | yes | cron, interval |
| AzureML | yes | cron, interval |
| Databricks | yes | cron |
| HyperAI | yes | cron, one-time |
| Kubeflow | yes | cron, interval |
| Kubernetes | yes | cron |
| Local | no | none |
| Local Docker | no | none |
| SageMaker | yes | cron, interval, one-time |
| SkyPilot (AWS / Azure / GCP / Lambda) | no | none |
| Tekton | no | none |
| Vertex | yes | cron |
