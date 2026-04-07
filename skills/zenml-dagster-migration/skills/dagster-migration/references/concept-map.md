# Dagster -> ZenML Concept Map

Complete mapping of Dagster concepts to their ZenML equivalents. Each entry is classified as **direct** (clean 1:1 mapping), **approximate** (conceptual equivalent with different semantics), or **absent** (no ZenML equivalent -- needs redesign).

## Table of Contents

- [Core Primitives](#core-primitives)
- [Data and IO](#data-and-io)
- [Configuration and Resources](#configuration-and-resources)
- [Control Flow and Execution](#control-flow-and-execution)
- [Infrastructure and Compute](#infrastructure-and-compute)
- [Dagster+ Features](#dagster-features)
- [Orchestrator scheduling support](#orchestrator-scheduling-support)

## Core Primitives

| Dagster Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `@asset` | `@step` output inside a `@pipeline` | approximate | Dagster assets are first-class data products with selection/materialization semantics. ZenML step outputs are artifacts produced during pipeline execution. Preserve compute logic, but explicitly decide pipeline boundaries first. |
| `@multi_asset` | Multi-output `@step` | approximate | Good fit when one compute body emits multiple outputs. If the Dagster code relies on subsetting individual outputs, always flag it. |
| `@graph_asset` | Helper steps inside a pipeline, with a terminal output artifact | approximate | Dagster exposes an asset while hiding internal ops. ZenML can model the internal steps directly, but the asset identity becomes an artifact naming convention. |
| `@graph_multi_asset` | Helper steps plus multi-output step or multiple terminal steps | approximate | Similar to `@graph_asset`, but watch for subset execution semantics. |
| `AssetSpec` | Artifact naming/config conventions | approximate | Useful as a naming and ownership hint, not as a runtime primitive. |
| `AssetKey` | Artifact name, tags, or `ArtifactConfig(name=...)` | approximate | Similar identity role, but ZenML artifacts are scoped to pipeline execution rather than an always-addressable asset graph. |
| `AssetIn` / `AssetOut` | Step inputs and outputs | approximate | Dependency intent maps cleanly, but storage/loading behavior is not separable from execution in the same way. |
| `Definitions` | Python module assembling pipelines, steps, configs, schedules | approximate | Both are composition roots, but ZenML does not expose a single registry object with Dagster's definitions semantics. |
| `@op` | `@step` | direct | Both wrap a unit of compute. Dagster ops often migrate more cleanly than assets because they already model execution units. |
| `@graph` | Composition helper or `@pipeline` | approximate | A graph often becomes either a ZenML pipeline or a helper function that wires steps. |
| `@job` | `@pipeline` | approximate | Strong fit for op/job-centric Dagster code. The mapping is nearly direct for op/job-centric Dagster code, but weaker for asset jobs because ZenML lacks asset-selection execution semantics. |
| Asset job / `define_asset_job` | One or more ZenML pipelines | approximate | The central migration decision: a Dagster asset job may need to become multiple ZenML pipelines if users rely on selective materialization or different schedules. |
| `AssetSelection` | Separate pipeline boundaries or explicit entry points | absent | ZenML does not have Dagster's first-class asset-selection and subset materialization model. |
| `SourceAsset` | `ExternalArtifact` or explicit source-loading step | approximate | Both represent external data, but the lifecycle and observability model differ. |
| Observable source asset | ExternalArtifact + validation/observation step | absent | ZenML can model checks and metadata logging, but not a first-class observable source asset concept. |
| `DynamicOut` / `DynamicOutput` | `@pipeline(dynamic=True)` with `.map()` or explicit redesign | approximate | Possible only when the target ZenML stack supports dynamic pipelines and the semantics are acceptable. Current ZenML docs describe support for `local`, `local_docker`, `kubernetes`, `sagemaker`, `vertex`, and `azureml`; verify current support and API details before committing to this path. |

## Data and IO

| Dagster Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `IOManager` | Artifact store + materializer + explicit source/sink step | approximate | Dagster IO managers can control both write and downstream load behavior. ZenML materializers primarily handle serialization/deserialization for artifacts; business logic around loading external systems usually belongs in steps. |
| `ConfigurableIOManager` | Materializer plus config/secrets/service connector wiring | approximate | Similar intent, but split across multiple ZenML primitives. |
| `FilesystemIOManager`, `mem_io_manager`, custom storage IO | Artifact store configuration or explicit storage step | approximate | File-based serialization often maps well; in-memory semantics and custom lazy-loading behavior do not, and should be flagged when they matter. |
| Materialization event | Artifact creation + metadata logging | approximate | ZenML artifacts are created automatically when a step returns a value. |
| Observation event | `log_metadata()` or validation step | approximate | Capture observations explicitly in a step; no first-class Dagster observation event. |
| Asset checks | Validation step consuming the artifact | approximate | Preserve the check logic, but the independently selectable asset-check execution model is absent. |
| Dagster types | Python type hints + materializers | approximate | ZenML type annotations drive materializer selection rather than Dagster runtime type checks. |
| Asset metadata | Artifact metadata/tags | approximate | Similar idea, different lifecycle and UI surfaces. |

## Configuration and Resources

| Dagster Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `Config` / Pydantic config | Pipeline parameters + YAML config | approximate | Similar validation intent, but ZenML passes typed Python values into steps/pipelines. |
| `RunConfig` | Pipeline invocation args / run configuration | approximate | Runtime values still exist, but the API and override semantics differ. |
| `EnvVar` | ZenML secrets or environment variables | approximate | Same purpose, different management surface. |
| `ConfigurableResource` | Step-local client object, ZenML secret, service connector, or stack component | approximate | Treat this as an intent-mapping exercise: infrastructure-wide concerns go to the stack; request-scoped clients can stay local to steps. |
| Resource dependency injection | Step helper construction or explicit function arguments | approximate | ZenML does not have the same resource injection system, so dependencies become explicit Python objects or stack-backed integrations. |

## Control Flow and Execution

| Dagster Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Asset dependency graph | Step invocation order with artifact wiring | approximate | Compute order can be preserved, but asset identity and partial-materialization semantics change. |
| Partition definitions (`DailyPartitionsDefinition`, etc.) | Pipeline parameters plus schedule/trigger discipline | approximate | Preserve the partition key as an explicit parameter; partition-aware orchestration semantics are not first-class in ZenML. |
| Partition mappings | Custom parameter plumbing or redesign | absent | No native equivalent for cross-partition dependency semantics. |
| Backfills | Repeated pipeline runs driven externally | approximate | Feasible, but operational behavior lives in triggering logic rather than as a Dagster-native backfill primitive. |
| Schedules | `Schedule(...)` on supported orchestrators | approximate | ZenML delegates schedules to the orchestrator rather than owning a scheduler itself. |
| Sensors | External trigger or polling step | absent | No sensor primitive, no cursor semantics, no lightweight deferral. |
| Asset sensors | External trigger or observation step plus trigger service | absent | Same issue as sensors, with extra asset-graph semantics missing. |
| Declarative Automation / auto-materialize policies | External automation logic + triggers | absent | The policy engine has no direct ZenML equivalent. Always flag. |
| Freshness policies | Explicit SLA/monitoring logic | absent | Can be approximated with schedules plus checks, but not as a first-class orchestration policy. |
| Asset reconciliation | Manual pipeline decomposition + trigger logic | absent | Requires redesign around pipeline boundaries and external automation. |
| Conditional branching based on runtime values | Dynamic pipeline or redesign | approximate | Only viable if dynamic pipelines are available and the branching semantics are acceptable. |
| Retry policy (`RetryPolicy`) | `StepRetryConfig` | direct | Retry intent maps cleanly. |
| Hooks / run-status monitoring | Step hooks, metadata logging, external monitoring | approximate | Useful for notifications and observability, but the operational surfaces differ. |

## Infrastructure and Compute

| Dagster Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Executor / launcher config | Orchestrator + step operator configuration | approximate | Same high-level concern, different APIs and abstractions. |
| Container resources / executor tags | `ResourceSettings` / `DockerSettings` | approximate | Preserve intent, not the exact runtime contract. |
| Docker executor / k8s run launcher | Containerized orchestrator or step operator | approximate | ZenML is Python-first and container-oriented, but the specific execution details differ. |
| Code location / workspace | Python project / code repository integration | approximate | Move Dagster repository structure into importable Python modules for ZenML. |
| Multi-code-location deployments | Multiple repos/modules or multiple pipelines | approximate | Supported operationally, but not with Dagster's same workspace model. |

## Dagster+ Features

| Dagster+ Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Hosted deployment / control plane | ZenML server / ZenML Pro | approximate | Both offer central control planes, but features and operational assumptions differ. |
| Alerting | Alerters, hooks, external monitoring | approximate | Comparable outcome, different configuration model. |
| Asset health and observability views | Artifact lineage, metadata, dashboards | approximate | ZenML is artifact/run-centric rather than asset-centric. |
| Branch deployments / branch environments | Separate stacks, workspaces, or code branches | approximate | Similar operational intent, different product surface. |

## Orchestrator scheduling support

ZenML scheduling support is orchestrator-dependent. Verify the target orchestrator before migrating any Dagster schedule-heavy project. Treat the table below as a planning aid, not a permanent guarantee of current support.

| Orchestrator | Scheduling | Types Supported |
|---|:---:|---|
| Airflow | Yes | Cron, Interval |
| AzureML | Yes | Cron, Interval |
| Databricks | Yes | Cron only |
| HyperAI | Yes | Cron, One-time |
| Kubeflow | Yes | Cron, Interval |
| Kubernetes | Yes | Cron only |
| Local | No | N/A |
| LocalDocker | No | N/A |
| SageMaker | Yes | Cron, Interval, One-time |
| SkyPilot | No | N/A |
| Tekton | No | N/A |
| Vertex | Yes | Cron only |
