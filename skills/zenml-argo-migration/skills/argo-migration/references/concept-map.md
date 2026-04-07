# Argo Workflows -> ZenML Concept Map

Complete mapping of Argo Workflows concepts to their ZenML equivalents. Each entry is classified as **direct** (clean mapping), **approximate** (conceptual equivalent with changed semantics), or **absent** (no portable ZenML equivalent -- requires redesign).

## Table of Contents

- [Core Primitives](#core-primitives)
- [Template Types](#template-types)
- [Data Passing](#data-passing)
- [Control Flow](#control-flow)
- [Execution Features](#execution-features)
- [Kubernetes-Native Features](#kubernetes-native-features)
- [Argo Events](#argo-events)
- [Orchestrator Scheduling Support](#orchestrator-scheduling-support)

## Core Primitives

| Argo Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `Workflow` | `@pipeline` plus a pipeline run | approximate | Same top-level role, but Argo `Workflow` is a YAML CRD + live status object; ZenML stores the definition in Python and tracks the run separately. |
| `WorkflowTemplate` | Reusable `@pipeline` in a Python module/package | approximate | Reuse exists in both systems, but Argo stores templates in-cluster while ZenML stores them in code. |
| `ClusterWorkflowTemplate` | Shared pipeline package, internal library, or snapshot | approximate | No cluster-global ZenML template resource. |
| `CronWorkflow` | `Schedule(...)` on a pipeline, or schedule trigger attached to a deployment/snapshot | approximate | Argo couples schedule + workflow spec; ZenML separates scheduling from the pipeline definition. |
| `entrypoint` | Top-level pipeline function body | direct | Both identify the root execution entry. |
| `templateRef` / `workflowTemplateRef` | Pipeline composition, shared code import, or external snapshot/deployment trigger | approximate | Reuse exists, but child-run semantics and operational boundaries often change. |
| Workflow-of-workflows pattern | Pipeline composition or external orchestration invoking another ZenML runnable | approximate | Treat run boundaries explicitly; plain composition may change retries and observability. |

## Template Types

| Argo Template Type | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `container` | `@step` with `DockerSettings`, sometimes `subprocess.run(...)` | approximate | ZenML still expects a Python-capable execution environment. |
| `script` | `@step` body for Python, or subprocess wrapper for shell | approximate | Clean for inline Python; less clean for arbitrary shell. |
| `dag` | Pipeline DAG built by step calls and artifact wiring | direct | Plain success-path DAGs map well; enhanced status expressions do not. |
| `steps` | Sequential Python calls and independent branches, often in a pipeline or dynamic pipeline | approximate | Same intent, different representation; no array-of-arrays syntax. |
| `resource` | Step that calls Kubernetes or cloud APIs explicitly | approximate | No first-class resource-template primitive exists, but the behavior can be rewritten as an explicit API step. |
| `suspend` | External approval gate, split pipeline, or version-sensitive wait/resume feature | absent | Do not treat as a clean portable replacement; if you explore `zenml.wait(...)`, require a recent 0.94.x release, preferably `>=0.94.1`. |
| `http` | Step using `requests` or another client SDK | approximate | Transport logic moves into Python code. |
| `plugin` | Explicit SDK/API step, webhook call, or external service | absent | No plugin-template analogue. |
| `containerSet` | Redesign as one step, supervisor process, or external service | absent | No multi-container same-pod primitive. |
| `data` | Loader/transform `@step`s and typed artifacts | approximate | ZenML handles data sourcing and transformation as ordinary step logic. |

## Data Passing

| Argo Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Workflow parameters | Pipeline function arguments | direct | Cleanest mapping. |
| Template input parameters | Step function arguments | direct | Cleanest mapping. |
| `arguments.parameters` on a task/step | Step invocation kwargs | direct | The dependency edge becomes a function call. |
| `{{workflow.parameters.foo}}` | Python variable / pipeline arg `foo` | direct | Replace string substitution with ordinary variables. |
| Output parameter via `valueFrom.path` | Named step return value | approximate | File extraction becomes an explicit return. |
| `outputs.result` from script/container/HTTP | Explicit step return value | approximate | ZenML has no implicit stdout/body result contract. |
| Input/output artifacts between templates | Upstream/downstream typed artifacts | approximate | Same idea, but ZenML artifacts are typed objects in an artifact store. |
| Artifact `path:` inside a template | Internal step implementation detail, or explicit `Path` / URI artifact | approximate | Do not assume downstream steps share the same filesystem. |
| Artifact repository / `artifactRepositoryRef` | Artifact store + service connectors + secrets | approximate | ZenML has no per-workflow artifact repo reference. |
| S3 / GCS / MinIO / Git / HTTP / HDFS artifact locations | Artifact store or explicit IO at pipeline edges | approximate | External URIs usually become loader/publisher steps. |
| `globalName` for outputs | Explicit pipeline outputs, metadata, model registry, or external store | approximate | No automatic workflow-global output namespace. |
| File-based `/tmp/...` contracts | Typed return values or explicit external storage URIs | approximate | Must not be preserved implicitly. |
| Pre-existing external data | `ExternalArtifact` or explicit loader step | approximate | ZenML can inject external data, but via a different mechanism. |

## Control Flow

| Argo Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `dag.tasks[].dependencies` | Artifact wiring and, when needed, explicit `after` ordering | direct | For plain success-only dependencies. |
| `dag.tasks[].depends` with status expressions | No direct equivalent | absent | `.Failed`, `.Errored`, `.Skipped`, `.Daemoned`, and boolean status logic require redesign. |
| Steps template outer arrays | Sequential Python step calls | direct | The structure becomes ordinary code. |
| Steps template inner arrays | Independent branches the orchestrator may run in parallel | approximate | Parallelism emerges from dependencies, not array syntax. |
| `when` on parameter/value expressions | `if` / `for` in `@pipeline(dynamic=True)` | approximate | Works for value-based control flow. |
| `when` on task status | No direct equivalent | absent | Failure-state branching is not a ZenML primitive. |
| `withItems` | Dynamic pipeline map or manual loop | approximate | Intent survives, but semantics and parallelism limits differ. |
| `withParam` | Upstream step returns a real list artifact, then dynamic fanout | approximate | JSON-string expansion becomes typed list expansion. |
| `withSequence` | Python `range(...)` in a dynamic pipeline | direct | Clean intent match. |
| `continueOn` | Typed result objects and careful use of pipeline execution modes | approximate | No per-edge direct equivalent. |
| `onExit` | Hooks, idempotent cleanup, external cleanup, or in-step `try/finally` | approximate | Not a guaranteed workflow-wide finalizer. |
| Lifecycle hooks | Step hooks / notifications | approximate | ZenML hooks are simpler and not expression-driven. |

## Execution Features

| Argo Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `retryStrategy.limit` + backoff | `StepRetryConfig(max_retries, delay, backoff)` | approximate | Similar shape, smaller policy surface. |
| `retryPolicy` (`OnFailure`, `OnError`, etc.) | Exception discipline inside step code | approximate | No first-class retry policy taxonomy. |
| Retry expression using `lastRetry.*` | No direct equivalent | absent | Must be rewritten in code or external control logic. |
| Workflow `activeDeadlineSeconds` | Orchestrator/job timeout settings, usually platform-specific | approximate | No portable global workflow timeout primitive. |
| Template timeout / deadline | In-code timeout or orchestrator-specific setting | approximate | Not a portable step-level primitive. |
| DAG `failFast` | Closest analogue is pipeline execution mode such as `STOP_ON_FAILURE` | approximate | Similar intent, not identical behavior. |
| `memoize` | Step caching | approximate | Argo uses explicit keys; ZenML hashes code + inputs + other factors. |
| `synchronization.mutexes` | External lock service or platform lock | absent | No built-in portable mutex primitive. |
| `synchronization.semaphores` | External concurrency control | absent | No built-in portable semaphore primitive. |
| Workflow/controller `parallelism` limits | Orchestrator/platform settings | approximate | Usually moves out of pipeline code. |

## Kubernetes-Native Features

| Argo Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Container `resources.requests/limits` | `ResourceSettings(...)` | approximate | Portable request API, but enforcement varies by orchestrator. |
| GPU resource scheduling | `ResourceSettings(gpu_count=...)` plus orchestrator settings | approximate | Placement details stay platform-specific. |
| `nodeSelector` | Kubernetes-aware orchestrator pod settings | approximate | Available only on supporting orchestrators. |
| `tolerations` | Kubernetes-aware orchestrator pod settings | approximate | Orchestrator-specific, not portable pipeline logic. |
| `affinity` | Orchestrator pod settings | approximate | Same portability warning. |
| `schedulerName` | Kubernetes orchestrator pod settings | approximate | Kubernetes-only. |
| `serviceAccountName` / RBAC | Stack auth, service connectors, orchestrator settings | approximate | Auth moves into stack configuration more than pipeline code. |
| Pod secrets / env / imagePullSecrets | ZenML secrets store + connectors + pod settings | approximate | Possible, but modeled differently. |
| `volumes` | No portable equivalent across isolated steps | absent | Prefer artifacts, external durable storage, or collapsing logic into one step. |
| `volumeClaimTemplates` | No direct equivalent | absent | Shared-workspace patterns must be redesigned. |
| `emptyDir` shared scratch space | No cross-step equivalent | absent | Only same-container or same-step redesign preserves it. |
| `sidecars` | No portable equivalent | absent | Same-pod concurrent helper containers do not map cleanly. |
| `initContainers` | Kubernetes orchestrator pod settings only | approximate | Possible only as Kubernetes-specific pod customization. |
| Daemon containers | No equivalent | absent | Background same-scope pod semantics are missing. |
| `podSpecPatch`, labels, env, additional pod args | Kubernetes orchestrator `pod_settings` | approximate | Powerful, but explicitly non-portable. |

## Argo Events

| Argo Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `EventSource` | External webhook / broker / queue feeding ZenML | absent | No native ZenML EventSource object is documented in current public docs; legacy event-source APIs were removed in 0.94.0. |
| `Sensor` | External automation glue or trigger-management flow | absent | No documented native dependency manager analogue for event conditions. |
| `Trigger` | Pro trigger concept for automated execution, or deployment / snapshot / pipeline run invocation | approximate | ZenML introduced a Pro `Trigger` concept in 0.94.0, but the first documented trigger type is schedules, not an Argo-Events-style graph. |
| Event dependency conditions | External event logic | absent | No Argo-style boolean event dependency graph is documented publicly. |
| Webhook event source | External webhook handler calling a ZenML endpoint | approximate | Integration pattern, not a first-class ZenML event source. |
| Schedule event source | ZenML `Schedule` or Pro schedule trigger | approximate | Time-based triggering is documented and supported when the orchestrator supports it. |
| EventBus / CloudEvents fabric | External platform concern | absent | No direct ZenML equivalent documented publicly. |

## Orchestrator Scheduling Support

Scheduling is orchestrator-dependent in ZenML. Keep this table nearby whenever migrating a `CronWorkflow`:

| Orchestrator | Scheduling Support | Supported Schedule Types | Native Schedule Management |
|---|:---:|---|:---:|
| AirflowOrchestrator | ✅ | Cron, Interval | ⛔️ |
| AzureMLOrchestrator | ✅ | Cron, Interval | ⛔️ |
| DatabricksOrchestrator | ✅ | Cron only | ⛔️ |
| HyperAIOrchestrator | ✅ | Cron, One-time | ⛔️ |
| KubeflowOrchestrator | ✅ | Cron, Interval | ⛔️ |
| KubernetesOrchestrator | ✅ | Cron only | ✅ |
| LocalOrchestrator | ⛔️ | N/A | N/A |
| LocalDockerOrchestrator | ⛔️ | N/A | N/A |
| SagemakerOrchestrator | ✅ | Cron, Interval, One-time | ⛔️ |
| SkypilotAWSOrchestrator | ⛔️ | N/A | N/A |
| SkypilotAzureOrchestrator | ⛔️ | N/A | N/A |
| SkypilotGCPOrchestrator | ⛔️ | N/A | N/A |
| SkypilotLambdaOrchestrator | ⛔️ | N/A | N/A |
| TektonOrchestrator | ⛔️ | N/A | N/A |
| VertexOrchestrator | ✅ | Cron only | ⛔️ |

Always verify the target orchestrator before promising a clean schedule migration.
