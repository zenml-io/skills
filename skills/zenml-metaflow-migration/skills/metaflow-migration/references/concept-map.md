# Metaflow -> ZenML Concept Map

Complete mapping of Metaflow concepts to their ZenML equivalents. Each entry is classified as **direct** (clean 1:1 mapping), **approximate** (similar intent but different semantics), or **absent** (no safe ZenML equivalent -- needs redesign).

## Table of Contents

- [Core Primitives](#core-primitives)
- [Data Passing and Parameters](#data-passing-and-parameters)
- [Control Flow](#control-flow)
- [Decorators](#decorators)
- [Infrastructure and Compute](#infrastructure-and-compute)
- [Outerbounds Extensions](#outerbounds-extensions)
- [Orchestrator Scheduling Support](#orchestrator-scheduling-support)

## Core Primitives

| Metaflow Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `FlowSpec` | `@pipeline`-decorated function | approximate | Metaflow graph structure lives in a class and transitions are expressed with `self.next(...)`; ZenML builds the DAG inside a pipeline function. |
| `@step` on a `FlowSpec` method | `@step` on a standalone function | approximate | Same decorator name, different object model. Metaflow mutates `self`; ZenML consumes inputs and returns outputs. |
| `self.next(...)` | Calling downstream steps in the pipeline function | approximate | Metaflow transitions are runtime graph declarations; ZenML is usually static unless using dynamic pipelines. |
| Reserved `start` step | First invoked ZenML step | absent | ZenML has no reserved `start` node. |
| Reserved `end` step | Final step(s) or pipeline exhaustion | absent | ZenML has no reserved `end` node. |
| Run | Pipeline run | direct | Both persist run metadata and artifacts, but the object model differs. |
| Task | Step run / mapped invocation | approximate | Metaflow exposes tasks explicitly, especially in `foreach`; ZenML mapped invocations are similar but not identical. |

## Data Passing and Parameters

| Metaflow Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `self.x = value` in a step | Step return value(s) | approximate | In Metaflow, assignment creates persisted artifacts automatically. In ZenML, only returned values become artifacts. |
| Reading `self.x` downstream | Step input parameter | approximate | ZenML requires the dependency to be explicit in the function signature. |
| Arbitrary Python artifact object | Typed artifact + materializer | approximate | ZenML serialization depends on type hints and materializers, not generic pickle-by-default behavior. |
| `Parameter` | Pipeline parameter | approximate | Both represent run inputs, but ZenML parameters are explicitly passed and typically JSON-serializable. |
| `JSONType` parameter | Dict/list pipeline parameter | direct | Both support structured JSON-like values. |
| `IncludeFile` | Registered/external artifact or explicit file ingestion | approximate | No exact "read file contents into the flow as a built-in parameter" primitive. |
| `Config` | YAML config, `.with_options(config_file=...)`, settings objects | approximate | Similar purpose, different abstraction and evaluation model. |
| `current` | `get_step_context()` | approximate | ZenML exposes step/run context, but not the full Metaflow `current.*` surface. |
| `metaflow.S3` | Artifact store + external artifacts + cloud SDKs | approximate | Same broad purpose, different APIs and lineage semantics. |
| Namespaces | Tags, separate servers, or ZenML Pro workspaces/projects | approximate | Similar isolation intent, different control plane. |
| Tags | Tags | approximate | Both support tagging; semantics and APIs differ. |

## Control Flow

| Metaflow Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Linear `self.next(self.a)` chain | Static pipeline composition | direct | Safest migration case. |
| `self.next(self.left, self.right)` | Parallel downstream step calls + explicit join step | approximate | ZenML requires the join contract to be explicit. |
| Join step `def join(self, inputs)` | Reducer/join step with explicit branch inputs | approximate | No special `inputs` object in ZenML. |
| `merge_artifacts(inputs)` | No direct equivalent | absent | Must write explicit merge/conflict logic. |
| `self.next(step, foreach="items")` | `@pipeline(dynamic=True)` + `.map()` | approximate | Similar intent, but dynamic pipelines are experimental and currently documented for `local`, `local_docker`, `kubernetes`, `sagemaker`, `vertex`, and `azureml`. |
| `self.input` | Mapped item input parameter | approximate | Pass the item explicitly into the mapped step. |
| `self.index` | Explicit index carried in the mapped payload | approximate | ZenML has no implicit foreach index object. |
| Conditional `self.next({...}, condition="x")` | Dynamic pipeline branching or explicit redesign | approximate | Runtime branching exists, but semantics differ, usually requires `.load()` for decisions, and the feature surface is narrower. |
| Recursion | Redesign into dynamic orchestration or repeated pipeline runs | absent | No documented general recursion primitive in ZenML. |

## Decorators

| Metaflow Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `@retry` | `StepRetryConfig(max_retries, delay, backoff)` | direct | Cleanest decorator mapping. |
| `@catch` | Explicit result/error envelope, alternate path, or redesign | absent | No direct "treat failure as successful continuation" primitive. |
| `@timeout` | Orchestrator-specific timeout config or in-step timeout logic | absent | No single portable core timeout decorator. |
| `@resources` | `ResourceSettings(cpu_count, memory, gpu_count, ...)` | approximate | Intent maps well; enforcement varies by backend. |
| `@conda` | `DockerSettings(requirements=..., parent_image=...)` | approximate | ZenML thinks in container images, not per-step Conda env creation. |
| `@pypi` | `DockerSettings(requirements=[...])` | approximate | Similar dependency intent, different execution model. |
| `@conda_base` | Pipeline-level `DockerSettings` | approximate | Shared dependency baseline at the pipeline level. |
| `@batch` | No direct equivalent | absent | Choose a target compute platform explicitly; do not claim a portable mapping. |
| `@kubernetes` | Kubernetes orchestrator or step operator settings | approximate | Similar destination, different control plane. |
| `@environment` | Environment variables in Docker/runtime settings | approximate | Similar intent, different configuration surface. |
| `@secrets` | ZenML secrets store + service connectors | approximate | Both manage credentials, but ZenML separates secrets from connector auth. |
| `@card` | Metadata logging, artifact visualizations, dashboard views | absent | No direct card/report canvas equivalent. |
| `@schedule` | `Schedule(...)` via `.with_options(schedule=...)` | direct | Scheduling exists, but support depends on orchestrator. |
| `@trigger` | External eventing + deployment or API trigger | absent | No decorator-level event trigger primitive. |
| `@trigger_on_finish` | External chaining / API-driven orchestration | absent | No direct built-in equivalent. |
| `@project` | Tags or ZenML Pro projects/workspaces | approximate | Similar grouping intent, different semantics. |
| `@checkpoint` | Explicit checkpoint artifacts or external durable storage | absent | No decorator-level equivalent for in-task recovery state. |
| Custom decorators | Hooks, wrapper steps, helper libraries, custom components | approximate | Translate by intent, not by syntax. |

## Infrastructure and Compute

| Metaflow Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Local runtime | Local orchestrator / direct pipeline call | approximate | Metaflow local runs isolate steps in separate processes; ZenML local execution is usually in the active environment. |
| `run --with kubernetes` | Use a Kubernetes-backed stack | approximate | Backend choice moves from CLI overlay to stack/runtime configuration. |
| `run --with batch` | No direct equivalent | absent | Redesign to the chosen remote compute target. |
| Step Functions / Argo deployment | Choose a ZenML orchestrator and deployment strategy | approximate | Similar operational goal, different deployment contract. |
| Object-store datastore | Artifact store | approximate | Similar broad role, different data model and APIs. |
| Metadata service / UI | ZenML server and dashboard | approximate | Similar observability goal, different object model. |
| `metaflow.client` | `zenml.client.Client` | approximate | Both expose lineage and run history, but traversal differs. |
| `Runner` | Direct pipeline invocation, SDK, CLI, snapshots | approximate | Similar programmatic execution intent. |
| `Deployer` | Deployments, snapshots, SDK/REST run triggers | approximate | Similar operational niche, different abstractions. |
| `resume` | Caching + explicit artifact/checkpoint reuse | approximate | Operationally similar, but not a semantic drop-in replacement. |
| CLI `--with <decorator>` overlays | Code/config/stack settings | absent | No universal ZenML equivalent to "attach decorators from the CLI". |

## Outerbounds Extensions

| Outerbounds Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| One-command production deploy | ZenML stacks, schedules, snapshots, or deployments | approximate | Similar goal, different control plane. |
| Fast Bakery | `DockerSettings`, image build settings, parent images | approximate | Same dependency-to-image story, different workflow. |
| Outerbounds `@docker` | `DockerSettings(parent_image=..., dockerfile=...)` | approximate | Similar intent, different API. |
| Compute pools / secure perimeters | Stack selection, step operators, service connectors | approximate | Infrastructure abstraction exists, but not as a single pool concept. |
| `@gpu_profile` | No direct equivalent | absent | Use experiment tracking, metadata, or external profilers instead. |
| Assets API / project assets | Artifacts + metadata + models | approximate | Similar storage intent, split across multiple ZenML concepts. |
| Long-running deployments / endpoints | Pipeline deployments or model deployers | approximate | Similar capability class, different lifecycle model. |
| Public `@model` decorator | No claimed mapping | absent | Do not invent a mapping unless the user provides concrete code and docs. |

## Orchestrator Scheduling Support

Check scheduling support before translating Metaflow schedules:

| Orchestrator | Scheduling | Types Supported |
|---|:---:|---|
| Kubernetes | Yes | Cron |
| Vertex | Yes | Cron |
| SageMaker | Yes | Cron, Interval, One-time |
| AzureML | Yes | Cron, Interval |
| Airflow | Yes | Cron, Interval |
| Kubeflow | Yes | Cron, Interval |
| Databricks | Yes | Cron only |
| Local / LocalDocker | No | -- |
| SkyPilot | No | -- |

Dynamic pipelines are also feature-limited by orchestrator. When a migration depends on `foreach` or runtime branching, call that out explicitly in the migration report. For manual dynamic loops, use `.load()` to make Python control-flow decisions and `.chunk(idx)` when you need to create DAG edges to individual elements instead of eagerly loading them.
