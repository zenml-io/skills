# SageMaker Pipelines → ZenML Concept Map

Complete mapping of SageMaker Pipelines concepts to their ZenML equivalents. Each entry is classified as **direct** (clean 1:1 mapping), **approximate** (conceptual equivalent with different semantics), or **absent** (no ZenML equivalent — needs redesign).

## Table of Contents

- [Core Concepts](#core-concepts)
- [Step Types](#step-types)
- [Data Passing and Parameters](#data-passing-and-parameters)
- [Control Flow and Scheduling](#control-flow-and-scheduling)
- [Infrastructure and AWS Integrations](#infrastructure-and-aws-integrations)
- [Model Management and Stack Components](#model-management-and-stack-components)

## Core Concepts

| SageMaker Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `Pipeline` | `@pipeline` | direct | Both define a workflow graph. ZenML authoring is Python-first rather than a SageMaker workflow DSL. |
| `PipelineSession` | No direct equivalent | absent | SageMaker uses it to capture `.fit()` / `.run()` calls into a pipeline definition. ZenML authoring does not require this layer. |
| `PipelineDefinitionConfig` | No direct equivalent | absent | ZenML does not expose a user-facing pipeline-definition compilation API. |
| `pipeline.upsert()` | Pipeline invocation on a configured stack | approximate | Similar intent, different lifecycle. ZenML manages deployment through the orchestrator. |
| `pipeline.start(parameters=...)` | Calling the pipeline with args or `.with_options(...)()` | approximate | Same high-level goal, different API surface. |
| `execution.wait()` | Normal client-side run waiting / tracking | approximate | ZenML run handling is not modeled as the same SageMaker execution object. |
| `PipelineExperimentConfig` | Experiment tracker component and/or explicit SageMaker experiment calls | approximate | Similar intent, different object model. |
| `ParallelismConfiguration` | Orchestrator/runtime-dependent parallelism | approximate | No single top-level ZenML equivalent with the same semantics. |
| `SelectiveExecutionConfig` | Caching plus explicit reruns / pipeline edits | approximate | Similar user intent, different implementation model. |

## Step Types

| SageMaker Step Type | ZenML Mapping | Mapping | Must Flag? | Notes |
|---|---|:---:|:---:|---|
| `ProcessingStep` | `@step` | direct | No | Usually the easiest migration. |
| `TrainingStep` | `@step` | direct | No | Keep the training logic in Python; decide separately whether to preserve SageMaker runtime settings. |
| `FailStep` | Step that raises an exception | approximate | No | Preserves intent, but no dedicated failure node exists in the ZenML graph. |
| `ConditionStep` | Python control flow, often `@pipeline(dynamic=True)` | approximate | Yes | Backend placeholder evaluation becomes Python logic. Dynamic pipelines are still experimental. |
| `TuningStep` | `@step` that launches HPO explicitly | approximate | Yes | Do not collapse to a single training run. |
| `TransformStep` | `@step` that launches Batch Transform explicitly | approximate | Yes | No native ZenML batch-transform primitive. |
| `ModelStep` | `@step` that performs explicit SageMaker model / registry work | approximate | Yes | Modern SageMaker path for create/register flows. |
| `CreateModelStep` | `@step` that calls SageMaker model APIs | approximate | Yes | Older API still appears in source pipelines. |
| `RegisterModel` | `@step` that calls SageMaker Model Registry APIs | approximate | Yes | Not the same as ZenML MCP. |
| `CallbackStep` | Explicit AWS integration step or redesign | absent | Yes | No native callback-token step in ZenML. |
| `LambdaStep` | Explicit boto3 Lambda call or redesign | absent | Yes | No native Lambda step; I/O and timeout semantics differ. |
| `QualityCheckStep` | `@step` that runs explicit quality-check logic | approximate | Yes | SageMaker baseline lifecycle is not preserved automatically. |
| `ClarifyCheckStep` | `@step` that runs Clarify or custom checks | approximate | Yes | Same caution as quality checks. |
| `EMRStep` | `@step` that submits EMR work | approximate | Yes | ZenML has no EMR step type. |
| `NotebookJobStep` | Rewrite notebook as code, or explicit notebook-job call | approximate | Yes | ZenML prefers Python steps over notebook-job orchestration. |
| `AutoMLStep` | `@step` that launches Autopilot / AutoML APIs | approximate | Yes | No native ZenML AutoML step primitive. |

## Data Passing and Parameters

| SageMaker Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `ParameterString` / `ParameterInteger` / `ParameterFloat` / `ParameterBoolean` | Typed pipeline parameters | direct | Usually a clean migration to Python arguments. |
| `ExecutionVariables` | Python values, step context, or logged metadata | approximate | No equivalent placeholder-expression system. |
| `Join` | Normal Python string formatting | approximate | Prefer explicit Python values over placeholder syntax. |
| `ProcessingInput` / `TrainingInput` | Artifact input or S3 channel settings | approximate | Use artifacts for idiomatic ZenML; keep S3 channels only when AWS-native semantics matter. |
| `ProcessingOutput` | Artifact output or S3 export settings | approximate | Same portability trade-off as inputs. |
| `TransformInput` | Explicit Batch Transform API args | approximate | No direct ZenML primitive. |
| Step `.properties` | Artifact values or metadata | absent | Placeholder property-path access is replaced, not preserved. |
| `PropertyFile` | Typed artifact returned from a step | absent | This is a data-model rewrite, not a shim. |
| `JsonGet` | Normal Python indexing on a loaded artifact | absent | Typically `metrics.load()[\"accuracy\"]` or a named field on a dataclass. |
| Large S3 URIs passed between steps | Artifact store or explicit URI artifacts | approximate | Keep raw URIs only when downstream AWS-native APIs genuinely require them. |

## Control Flow and Scheduling

| SageMaker Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `ConditionStep` | `@pipeline(dynamic=True)` + Python `if` / `else` | approximate | Use `.load()` only for small control artifacts. |
| `ConditionEquals` / `ConditionGreaterThan` / `ConditionLessThan` / related condition types | Python boolean expressions | approximate | Same logical intent, different execution environment. |
| `ConditionOr` / `ConditionNot` | `or` / `not` | approximate | Same caution: Python runtime instead of backend placeholder evaluation. |
| `depends_on` | Natural dependencies from artifact wiring or explicit control code | approximate | ZenML derives dependencies from data flow more often than from explicit graph declarations. |
| `PipelineSchedule` / `put_triggers()` | `Schedule(...)` + `.with_options(schedule=...)()` | approximate | Similar goal, different API surface. |
| Update schedule in place | No native SageMaker schedule management in ZenML | absent | Re-running a scheduled SageMaker pipeline creates a new schedule rather than updating the old one. |
| Dynamic pipeline scheduling on SageMaker orchestrator | Not supported | absent | Important migration caveat for `ConditionStep`-style pipelines. |

### Scheduling on the SageMaker Orchestrator

| Orchestrator | Scheduling Support | Supported Types | Native Schedule Management |
|---|:---:|---|:---:|
| `SagemakerOrchestrator` | Yes | Cron, Interval, One-time | No |

**Important caveat:** dynamic pipelines cannot currently be scheduled on the SageMaker orchestrator. If the migrated pipeline needs runtime branching **and** native SageMaker-backed scheduling, flag this immediately.

## Infrastructure and AWS Integrations

| SageMaker / AWS Concern | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Execution role | Orchestrator `execution_role` | direct | Core runtime mapping when keeping SageMaker. |
| Scheduler role | Orchestrator `scheduler_role` | direct | Needed for scheduled SageMaker-backed runs. |
| Client AWS auth | Service connector, explicit credentials, or implicit AWS profile | direct | Service connector is the recommended ZenML pattern. |
| Instance type / volume / environment | `SagemakerOrchestratorSettings` | direct | These are runtime settings, not business-logic parameters. |
| Warm pools | `keep_alive_period_in_seconds` | direct | Useful, but multichannel output or output modes other than `EndOfJob` make warm pools unavailable. |
| Force Training-step path | `use_training_step=True/False` | direct | Preserve intentionally; do not scatter this across business logic. |
| S3 import channels | `input_data_s3_uri`, `input_data_s3_mode` | direct | Good for "keep SageMaker" migrations. |
| S3 export channels | `output_data_s3_uri`, `output_data_s3_mode` | direct | Multichannel output or output modes other than `EndOfJob` prevent using TrainingStep and warm pools. |
| Pipeline tags / job tags | `pipeline_tags`, `tags` | direct | Good keep-SageMaker runtime mapping. |
| `CallbackStep` / SQS workflows | Explicit step code or external orchestration | absent | Requires a deliberate protocol design. |
| `LambdaStep` | Explicit step code calling Lambda | absent | Preserve the AWS dependency explicitly if kept. |
| SageMaker Experiments | Experiment tracker or explicit SageMaker experiment APIs | approximate | Similar goal, different semantics. |
| SageMaker Feature Store | Keep explicit SageMaker Feature Store usage, or build a custom integration | absent | Do not present the ZenML feature-store abstraction as a practical SageMaker Feature Store replacement. |
| Clarify / Model Monitor | Explicit SageMaker service calls inside steps | approximate | No first-class ZenML step type for these services. |

## Model Management and Stack Components

| SageMaker Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `CreateModelStep` | Explicit SageMaker model step or redesign | approximate | Keep AWS-native semantics only when you need them. |
| `RegisterModel` | Explicit SageMaker registry step or ZenML MCP | approximate | Choose the source of truth intentionally. |
| `ModelStep` | Explicit SageMaker model / registry step | approximate | Preferred modern SageMaker SDK path for some workflows. |
| Model Package Group | Explicit SageMaker registry calls | approximate | There is no 1:1 ZenML object. |
| Approval statuses | ZenML model stages / approvals, or explicit SageMaker approvals | approximate | Similar governance intent, different lifecycle semantics. |
| Hosted endpoint deployment | Explicit SageMaker endpoint APIs or a deliberate ZenML serving choice | approximate | Do not hide this design choice. |
| ZenML Model Control Plane | No native SageMaker Pipelines equivalent | direct on ZenML side | This is additive capability, not a registry shim. |
