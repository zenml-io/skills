# Vertex AI Pipelines / KFP v2 → ZenML Concept Map

Complete mapping of **Vertex AI Pipelines / KFP v2 / PipelineJob workflows** to ZenML. Each entry is classified as **direct** (clean 1:1 mapping), **approximate** (similar intent, different semantics), or **absent** (no ZenML equivalent — redesign required).

## Table of Contents

- [Core Pipeline](#core-pipeline)
- [Artifact Types](#artifact-types)
- [Control Flow](#control-flow)
- [Google Cloud Pipeline Components](#google-cloud-pipeline-components)
- [Scheduling and Triggers](#scheduling-and-triggers)
- [Infrastructure and Resources](#infrastructure-and-resources)
- [Vertex AI Platform Features](#vertex-ai-platform-features)

## Core Pipeline

| KFP / Vertex concept | ZenML equivalent | Mapping | Notes |
|---|---|:---:|---|
| `@dsl.pipeline` | `@pipeline` | direct | Both define the pipeline graph in Python. |
| typed pipeline parameters | typed pipeline function arguments | direct | This part migrates cleanly. |
| `@dsl.component` | `@step` | approximate | Similar shape, different execution and artifact semantics. |
| `@dsl.component(base_image=..., packages_to_install=...)` | `@step(settings={"docker": DockerSettings(...)})` | approximate | Same intent, different build/runtime model. |
| `@dsl.component(target_image=...)` | `DockerSettings(parent_image=...)` or a prebuilt image workflow | approximate | The image toolchain is different. |
| `@dsl.container_component` | wrapper `@step` plus containerization logic | approximate | Safe only when the container contract is simple and explicit. |
| `ContainerSpec` | wrapper step + ZenML container settings | approximate | Redesign-heavy when the original component is not really a Python function. |
| `Input[T]` / `Output[T]` | typed step inputs and return values | approximate | KFP types platform artifacts; ZenML types Python values. |
| `InputPath` / `OutputPath` | explicit file handling, `ExternalArtifact`, `UnmaterializedArtifact`, or a custom materializer | approximate | Path identity is not a first-class ZenML step contract. |
| `dsl.importer(...)` | `ExternalArtifact(...)` or explicit artifact lookup / registration | approximate | Similar intent, different mechanism. |
| `PipelineTask.after()` | explicit ordering or artifact dependency | approximate | Artifact edges remain the preferred dependency signal. |
| `compiler.Compiler().compile()` | none | absent | ZenML does not require user-facing compilation. |
| compiled IR YAML / JSON | none | absent | ZenML snapshots are not KFP pipeline templates. |
| placeholder helpers such as `PIPELINE_ROOT_PLACEHOLDER` | explicit config or artifact APIs | absent | Placeholder-driven path injection does not have a direct DSL match. |

## Artifact Types

| KFP artifact type | ZenML equivalent | Mapping | Notes |
|---|---|:---:|---|
| `Artifact` | custom Python type plus materializer | approximate | Use the real contract, not a fake generic placeholder. |
| `Dataset` | `pd.DataFrame`, dataset class, `Path`, or URI/reference object | approximate | Do not flatten URI-aware datasets into DataFrames blindly. |
| `Model` | model object artifact or explicit model-upload step | approximate | ZenML model artifacts are not Vertex Model Registry resources. |
| `Metrics` | `dict[str, float]` and/or metadata logging | approximate | Same goal, different API. |
| `ClassificationMetrics` | structured Python object plus metadata logging | approximate | Preserve the schema explicitly. |
| `SlicedClassificationMetrics` | structured Python object plus metadata logging | approximate | Keep slice structure stable and explicit. |
| `HTML` | `str`, `Path`, or a visualizable custom artifact | approximate | Validate UI rendering expectations after migration. |
| `Markdown` | `str`, `Path`, or a visualizable custom artifact | approximate | Same caution as HTML. |
| `InputPath(T)` / `OutputPath(T)` | explicit temp-file handling or a location-aware artifact contract | approximate | Treat as a contract decision, not a decorator rename. |

## Control Flow

| KFP / Vertex concept | ZenML equivalent | Mapping | Notes |
|---|---|:---:|---|
| `dsl.Condition` | `@pipeline(dynamic=True)` plus Python branching on `.load()` | approximate | Similar intent, different execution model. |
| `dsl.If` / `dsl.Elif` / `dsl.Else` | dynamic-pipeline branching | approximate | Safe only when you preserve runtime-vs-static semantics explicitly. |
| `dsl.OneOf` | selector step or ordinary Python branching | approximate | No dedicated ZenML primitive. |
| `dsl.ParallelFor` | `.map()` in a dynamic pipeline | approximate | Similar fan-out, different semantics and limitations. |
| `dsl.Collected` | reducer step over mapped outputs | approximate | Similar fan-in, different runtime model. |
| artifact-driven loops | `.load()` for control flow and `.map()` for fan-out | approximate | ZenML separates control decisions from artifact edges. |
| explicit async submission patterns | `.submit()` where appropriate | approximate | Use sparingly and document why. |
| `dsl.ExitHandler` | none | absent | Redesign cleanup / notification semantics explicitly. |
| final-status driven cleanup semantics | hooks, idempotent cleanup, or external failure handling | absent | Not a 1:1 pipeline primitive in ZenML. |

## Google Cloud Pipeline Components

All GCPC families should be treated as **rewrite-only** surfaces.

| GCPC family / example op | ZenML migration target | Mapping | Notes |
|---|---|:---:|---|
| BigQuery (`BigqueryQueryJobOp`) | `@step` calling BigQuery client / SQL API | absent | Rewrite as ordinary Python step code. |
| AutoML (`AutoMLTabularTrainingJobRunOp`) | `@step` calling Vertex AI SDK | absent | No ZenML operator parity. |
| Model upload (`ModelUploadOp`) | explicit SDK-calling upload step | absent | ZenML model artifacts are not Vertex Model resources. |
| Endpoint create / deploy (`EndpointCreateOp`, `ModelDeployOp`) | explicit SDK-calling deployment step | absent | Rewrite the cloud action explicitly. |
| Batch prediction | explicit SDK-calling step | absent | Preserve intent, not operator syntax. |
| Dataflow (`DataflowPythonJobOp`) | explicit Beam/Dataflow launch step | absent | This is orchestration over a service, not a native ZenML operator. |
| Dataproc jobs | explicit Dataproc API step | absent | Same rewrite rule. |
| hyperparameter tuning ops | explicit SDK-calling tuning step | absent | Model as step code or split into separate orchestration. |

## Scheduling and Triggers

| KFP / Vertex concept | ZenML equivalent | Mapping | Notes |
|---|---|:---:|---|
| `PipelineJob.create_schedule(...)` | `Schedule(cron_expression=..., start_time=..., end_time=...)` | approximate | ZenML exposes the common subset, not full lifecycle parity. |
| timezone-prefixed cron | `Schedule(cron_expression="TZ=... ...")` | approximate | Validate the exact cron form against your target orchestrator. |
| `max_run_count` / `max_concurrent_run_count` / queueing knobs | none | absent | These are not first-class ZenML schedule settings. |
| schedule update / delete via Vertex APIs | manual management on the orchestrator side | absent | ZenML does not fully manage Vertex schedule lifecycle. |
| template-path-based scheduling | none | absent | ZenML schedules pipelines, not compiled KFP templates. |

### Orchestrator scheduling support

| Orchestrator | Scheduling | Types Supported | Native schedule management |
|---|:---:|---|:---:|
| Airflow | Yes | Cron, Interval | No |
| AzureML | Yes | Cron, Interval | No |
| Databricks | Yes | Cron only | No |
| Kubeflow | Yes | Cron, Interval | No |
| Kubernetes | Yes | Cron only | Yes |
| SageMaker | Yes | Cron, Interval, One-time | No |
| Vertex | Yes | Cron only | No |
| Local / LocalDocker | No | — | — |

## Infrastructure and Resources

| KFP / Vertex mechanism | ZenML mechanism | Mapping | Notes |
|---|---|:---:|---|
| `set_cpu_limit`, `set_memory_limit`, `set_accelerator_limit` | `ResourceSettings(cpu_count=..., memory=..., gpu_count=...)` | approximate | Start with portable intent first. |
| `set_accelerator_type` | Vertex-specific orchestrator settings | approximate | Keep backend-specific details separate from business logic. |
| `add_node_selector_constraint()` | pod / orchestrator settings | approximate | Placement semantics are backend-specific. |
| `base_image` / `packages_to_install` | `DockerSettings(...)` | approximate | Same problem solved through a different build system. |
| `target_image` | prebuilt image or Docker parent image settings | approximate | Different toolchain boundary. |
| service account on `PipelineJob.run(...)` | Vertex orchestrator settings / service connector-backed auth | direct | Supported, but configured differently. |
| network / reserved IP ranges | Vertex-specific custom job parameters | approximate | These stay backend-specific. |
| persistent resources | Vertex-specific custom job parameters | direct | Supported, but only on the Vertex path. |
| `pipeline_root` | artifact store configuration in the active stack | approximate | Prefer stack config over hardcoded pipeline paths. |
| KFP / Vertex caching flags | ZenML step / pipeline caching | approximate | Same goal, different cache identity model. |

## Vertex AI Platform Features

| Vertex AI feature | ZenML mapping | Mapping | Notes |
|---|---|:---:|---|
| managed execution on Vertex | Vertex orchestrator | direct | This is the main keep-Vertex migration path. |
| Vertex run UI URL | orchestrator metadata / run links | approximate | Preserved in practice, but through ZenML metadata. |
| Vertex Experiments | Vertex experiment tracker integration plus explicit logging | approximate | Similar intent, different wiring. |
| TensorBoard integration | explicit experiment-tracker / SDK setup | approximate | Not automatic from ordinary artifact logging. |
| Vertex Model Registry | explicit upload / deployment steps | approximate | Must be added intentionally. |
| Vertex ML Metadata | ZenML metadata store + underlying Vertex metadata plane | approximate | Different canonical metadata systems. |
| pipeline templates / template registry | none | absent | ZenML has no 1:1 template-registry concept. |
| GCPC launch optimizations | none | absent | Rewritten SDK-calling steps can change startup / cost characteristics. |
