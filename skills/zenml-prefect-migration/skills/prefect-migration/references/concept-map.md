# Prefect → ZenML Concept Map

Complete mapping of Prefect concepts to their ZenML equivalents. Each entry is classified as **direct**, **approximate**, or **absent**.

## Table of Contents

- [Core Concepts](#core-concepts)
- [State Handling](#state-handling)
- [Execution and Concurrency](#execution-and-concurrency)
- [Data, Results, and Caching](#data-results-and-caching)
- [Configuration and Secrets](#configuration-and-secrets)
- [Deployment and Automation](#deployment-and-automation)
- [Orchestrator Scheduling Support](#orchestrator-scheduling-support)

## Core Concepts

| Prefect Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `@flow` | `@pipeline` | approximate | Both decorate Python callables, but Prefect flow bodies execute imperatively while ZenML static pipelines compile a DAG first. |
| `@task` | `@step` | direct | This is the cleanest migration path. |
| Direct task call inside flow | Step call inside pipeline | approximate | Code shape looks similar, but Prefect executes now while ZenML is composing the graph. |
| Nested flow / subflow | Pipeline composition | approximate | Structural reuse maps well; child-run/state semantics do not. |
| Flow return value | Pipeline outputs / artifacts | approximate | ZenML centers on artifacts and run metadata rather than an ordinary Python return flow. |
| `flow_run_name` | `with_options(run_name=...)` | approximate | Similar user-facing intent, different lifecycle semantics. |
| Flow timeout | Orchestrator/job timeout settings | approximate | Timeout lives in different places depending on ZenML stack/orchestrator. |

## State Handling

| Prefect Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `State` object | No first-class in-pipeline state object | absent | ZenML has run/step statuses, not an inspectable `State` passed through workflow code. |
| `Completed`, `Failed`, `Paused`, `Suspended`, `Cancelled`, etc. | Run/step execution status | approximate | Status exists in both systems, but the programmatic model is much richer in Prefect. |
| `return_state=True` | None | absent | Requires redesign. |
| Returning `Failed(...)` manually | Exception or explicit failure-as-data artifact | approximate | Usually redesign around explicit outputs instead of hidden state semantics. |
| `allow_failure()` | None | absent | No dependency-level equivalent in ZenML. |
| `on_completion`, `on_failure` | `on_success`, `on_failure` hooks | approximate | Hook model exists, but semantics and context differ. |
| `pause_flow_run()` | Dynamic wait / approval pattern / split workflow | approximate | Partial substitute only; not a drop-in orchestration-state match. |
| `suspend_flow_run()` | Dynamic wait + resume-compatible setup / split workflow | approximate / absent | Often redesign-required for durable resume semantics. |

## Execution and Concurrency

| Prefect Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Static flow code | Static pipeline | approximate | Safe only when workflow shape is known before execution. |
| Runtime branch on task output | `@pipeline(dynamic=True)` | approximate | Closest equivalent, but dynamic pipelines are still experimental and have limits. |
| `.submit()` | `step.submit()` in dynamic pipelines | approximate | Similar concurrency intent; failure handling differs. |
| `.map()` | `step.map()` in dynamic pipelines | approximate | Same-run artifact limitation and different orchestration semantics. |
| `ThreadPoolTaskRunner` | Dynamic `runtime="inline"` or orchestrator concurrency | approximate | Similar concurrency goal, different abstraction. |
| `ProcessPoolTaskRunner` | `runtime="isolated"` or isolated step execution | approximate | Similar isolation goal, different execution model. |
| `DaskTaskRunner` / `RayTaskRunner` | Run Dask/Ray inside a step or re-architect infra | approximate | No native ZenML task-runner analogue. |
| `wait_for=` dependency-only ordering | `after=` / explicit dependency | direct | One of the cleanest execution-control mappings. |
| Global `concurrency("name")` | None | absent | Must redesign with external locking or platform controls. |
| `rate_limit("name")` | None | absent | Must redesign with external quota or serialized logic. |
| Deployment/work-pool concurrency limit | Orchestrator or job concurrency controls | approximate | Operational goal overlaps, but API/scope differ. |

## Data, Results, and Caching

| Prefect Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Task/flow result | Artifact output | approximate | Both persist outputs, but artifacts are central in ZenML. |
| `persist_result=True` | Default artifact persistence | approximate | Similar intent, different default model and storage lifecycle. |
| `result_storage` | Artifact store | approximate | Same broad concern, different abstraction boundary. |
| `result_serializer` | Materializer | approximate | Closest equivalent. |
| `result_storage_key` | Artifact naming / URI / versioning | approximate | No exact 1:1 field. |
| Step caching | Step caching | approximate | Both have reuse semantics, but triggers/configuration differ. |
| `cache_key_fn` | None | absent | Custom cache-key callbacks do not have a direct ZenML equivalent. |
| Cache expiration / TTL | No exact equivalent | absent / approximate | Flag when business logic depends on the TTL itself. |
| Human-facing Prefect artifacts | ZenML artifacts / reports / dashboard metadata | approximate | Different UI and data model emphasis. |

## Configuration and Secrets

| Prefect Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Block system overall | Secrets + service connectors + stack config + YAML + params | approximate | Must split by concern; there is no single equivalent. |
| Secret Block | ZenML secret | direct | Closest clean mapping. |
| Typed config Block | YAML config / Pydantic config / stack settings | approximate | Same purpose, different storage/distribution model. |
| Service/auth Block | Service connector | approximate | Best mapping for cloud credentials and auth intent. |
| Variable | YAML config / pipeline parameters / secrets | approximate | Prefer explicit typing and separate secret handling. |
| `prefect.yaml` | ZenML YAML configuration | approximate | Similar configuration role, different schema and runtime contract. |
| Custom Block | Custom config model + secrets + infra config | approximate | Usually needs decomposition and redesign. |

## Deployment and Automation

| Prefect Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Deployment | Schedule + orchestrator + run config + sometimes snapshots | approximate | Map by intent, not by name. |
| `flow.serve()` | Local scheduled execution / deployment registration pattern | approximate | Similar convenience, different implementation model. |
| Work pool | Orchestrator / stack / execution backend | approximate | No single work-pool routing abstraction in ZenML OSS. |
| Process work pool | Local orchestrator | approximate | Similar local execution goal. |
| Docker work pool | LocalDocker / containerized orchestrator path | approximate | Similar environment shape, different control plane. |
| Kubernetes work pool | Kubernetes orchestrator | approximate | Closest clean infrastructure mapping. |
| Vertex work pool | Vertex orchestrator | approximate | Similar remote managed execution goal. |
| Push / managed work pools | None direct | absent / approximate | Usually needs external platform or ZenML Pro-specific redesign. |
| Worker | Orchestrator execution environment | approximate | Operationally similar, not the same API/control-plane concept. |
| Automations | ZenML Pro triggers or external automation | approximate / absent | No direct OSS counterpart. |
| Event-driven triggers | External eventing / triggers / webhooks | approximate | Depends on stack and product tier. |
| Pipeline deployment (ZenML) | No true Prefect equivalent | absent | ZenML pipeline deployments are HTTP services, not batch-run deployment configs. |

## Orchestrator Scheduling Support

Not all ZenML orchestrators support schedules. Check the target stack before claiming a clean schedule migration.

| Orchestrator | Scheduling Support | Types Supported |
|---|:---:|---|
| Kubernetes | Yes | Cron |
| Vertex | Yes | Cron |
| SageMaker | Yes | Cron, Interval, One-time |
| AzureML | Yes | Cron, Interval |
| Airflow (as ZenML orchestrator) | Yes | Cron, Interval |
| Kubeflow | Yes | Cron, Interval |
| Databricks | Yes | Cron only |
| Local / LocalDocker | No | — |
| SkyPilot | No | — |
