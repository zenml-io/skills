# Databricks Workflows -> ZenML Concept Map

Complete mapping of Databricks Workflows (Lakeflow Jobs) concepts to their ZenML equivalents. Each entry is classified as **direct** (clean 1:1 mapping), **approximate** (conceptual equivalent with different semantics), or **absent** (no ZenML equivalent -- needs redesign).

## Table of Contents

- [Orchestration Primitives](#orchestration-primitives)
- [Task Types](#task-types)
- [Control Flow and Scheduling](#control-flow-and-scheduling)
- [Data Passing and Parameters](#data-passing-and-parameters)
- [Error Handling and Notifications](#error-handling-and-notifications)
- [Infrastructure and Compute](#infrastructure-and-compute)
- [Stack Component Mappings](#stack-component-mappings)

## Orchestration Primitives

| Databricks Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Job (multi-task) | `@pipeline`-decorated function | direct | Both define a DAG. Databricks DAG is explicit JSON `tasks[]`; ZenML DAG is constructed by calling step objects inside the pipeline function. |
| Single-task job | Single-step pipeline | direct | Databricks "format" distinction is API-level; ZenML always supports multiple steps. |
| Task (`task_key`, task payload) | `@step`-decorated callable | direct | `task_key` maps to step function name. Repeated invocations are auto-suffixed in ZenML unless an explicit ID is set. |
| `depends_on` (explicit edges) | Implicit via passing step outputs to inputs | direct (data deps), approximate (ordering-only) | ZenML dependencies are induced by artifact edges. Pure ordering dependencies with no data flow require a synthetic artifact or control dependency pattern. |
| `depends_on` with `outcome: "true"/"false"` | Dynamic pipeline branching | approximate | Outcome-conditional edges require `@pipeline(dynamic=True)` in ZenML; see `condition_task` below. |

## Task Types

| Databricks Task Type | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `notebook_task` | `@step` wrapping refactored notebook logic | approximate | Notebooks rarely translate 1:1 due to global state, magics (`%sql`, `%pip`, `%run`), Spark session assumptions, and `dbutils` usage. Extract notebook code into plain Python modules. |
| `spark_python_task` (Python script) | `@step` executing the script logic | approximate | Databricks passes CLI-style args; ZenML prefers function parameters and typed artifacts. Keep CLI semantics only when necessary. |
| `python_wheel_task` | `@step` calling the wheel entry point as a Python function | approximate | Databricks runs an installed wheel using `package_name` + `entry_point`. In ZenML, import and call internal functions directly; use `DockerSettings` for dependencies. |
| `sql_task` | `@step` executing SQL via explicit client | approximate | Databricks SQL tasks use `warehouse_id` and `{{parameter_key}}` substitution. ZenML needs an explicit database connector and ZenML secrets for credentials. |
| `dbt_task` | `@step` running dbt CLI in a container | approximate | Databricks injects auth token; ZenML needs explicit credentials via secrets and network access configuration. |
| `spark_jar_task` | `@step` submitting JAR to Spark backend | approximate | ZenML is Python-first; JAR execution requires a custom step operator or external launcher. |
| `spark_submit_task` | `@step` calling `spark-submit` or Spark step operator | approximate | Not first-class in ZenML; achievable via step operator or subprocess wrapper. |
| `condition_task` (if/else) | Parameter-based branching or `@pipeline(dynamic=True)` | approximate | Databricks evaluates string operands without compute. ZenML needs a step or parameter to supply the value. Dynamic pipelines require orchestrator support. |
| `for_each_task` | `@pipeline(dynamic=True)` + `.map()` | approximate | Databricks iterates over a JSON array with explicit concurrency limits. ZenML fan-out parallelism is orchestrator-dependent. |
| `run_job_task` | Pipeline composition or API-triggered pipeline run | approximate | For code reuse within same repo: pipeline composition. For cross-workspace orchestration: trigger run via ZenML Client API. |
| `pipeline_task` (Lakeflow/DLT) | Convert to ZenML pipeline or external trigger step | approximate | Databricks `pipeline_task` triggers Declarative Pipelines. Either convert that pipeline to ZenML (preferred) or create a step that triggers it via API. |

## Control Flow and Scheduling

| Databricks Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `run_if: ALL_SUCCESS` (default) | Default ZenML behavior | direct | Steps run when their input artifacts are available (implying upstream success). |
| `run_if: ALL_DONE` | Pipeline execution mode + explicit failure handling | absent | ZenML has pipeline-level `execution_mode=CONTINUE_ON_FAILURE` but not per-step `run_if`. Emulation requires wrapping upstream steps to capture failures. **Always flag.** |
| `run_if: AT_LEAST_ONE_FAILED` | `on_failure` hooks or status-checking step | absent | No direct equivalent. Use hooks on upstream steps instead of a downstream recovery task. **Always flag.** |
| `run_if: NONE_FAILED` | Approximate via default behavior | approximate | Default ZenML behavior is similar (run if inputs available), but semantics differ when some upstream steps are skipped. |
| Cron scheduling (`quartz_cron_expression`) | `Schedule(cron_expression=...)` | approximate | Databricks uses 6-field Quartz cron with timezone; ZenML uses standard 5-field cron. Drop the seconds field. Scheduling is orchestrator-dependent. |
| Periodic trigger (`trigger.periodic`) | `Schedule(interval_second=...)` | approximate | Databricks periodic trigger has explicit interval + unit. ZenML supports interval seconds; enforcement is orchestrator-dependent. |
| File arrival trigger | External eventing + pipeline trigger | absent | Databricks integrates with Unity Catalog file events. ZenML OSS has no native file arrival trigger. Requires cloud events -> webhook -> pipeline run. **Always flag.** |
| Table update trigger | External eventing + pipeline trigger | absent | Similar to file arrival. No ZenML OSS equivalent. |
| Continuous jobs | Long-running service or recurring trigger | absent | Databricks continuous mode auto-restarts. ZenML has pipeline deployments for persistent execution, but streaming is typically a separate service. **Always flag.** |

### Orchestrator scheduling support

Not all ZenML orchestrators support scheduling. Check this before migrating scheduled jobs:

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

## Data Passing and Parameters

| Databricks Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `dbutils.jobs.taskValues.set(key, value)` | Step return value -> artifact | direct | Return the value from the step function. ZenML artifacts are typed and persisted. |
| `dbutils.jobs.taskValues.get(task_name, key)` | Step input parameter (artifact wiring) | direct | No explicit `get` needed -- pass the upstream step's output as a downstream input in the pipeline function. |
| `{{tasks.<task>.values.<key>}}` (dynamic ref) | Direct function-call wiring in pipeline | direct | Replace string substitution with actual artifact passing between steps. |
| `{{job.parameters.*}}` | Pipeline function parameters | approximate | Databricks replaces with string literal; ZenML parameters are typed Python values. Must parse and type-cast. |
| `dbutils.widgets.get()` / `dbutils.widgets.text()` | Step function parameters | approximate | Widgets are notebook-runtime API. In ZenML, the step signature defines parameters; values come from pipeline call or YAML config. |
| `{{job.start_time.*}}` / `{{job.run_id}}` | Compute in step or use `get_step_context()` | approximate | Databricks provides job-level metadata via dynamic refs. ZenML provides step/run context for metadata. |
| `base_parameters` on notebook tasks | Step function parameters | approximate | Databricks pushes params into notebooks as widget values. In ZenML, pass as regular function arguments. |
| `named_parameters` on wheel tasks | Step function parameters | approximate | Databricks passes CLI-style named args to wheel entry point. In ZenML, import the function and call with Python args. |
| SQL `{{parameter_key}}` substitution | Parameterized query in step code | approximate | Replace SQL template vars with parameterized queries using the SQL connector library. |
| Job-level `parameters[]` definition | Pipeline function signature | direct | Each Databricks job parameter becomes a typed pipeline parameter. |

## Error Handling and Notifications

| Databricks Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `max_retries` | `StepRetryConfig(max_retries=N)` | direct | Step-level retry configuration. |
| `min_retry_interval_millis` | `StepRetryConfig(delay=N)` (seconds) | direct | Convert from milliseconds to seconds. |
| `retry_on_timeout` | Application-level timeout wrapper | approximate | ZenML has no universal per-step timeout. If `retry_on_timeout=true`, the timeout + retry combo needs orchestrator-specific enforcement or an in-step wrapper. |
| `timeout_seconds` per task | Orchestrator-specific timeout or in-step wrapper | approximate | No uniform cross-orchestrator per-step timeout. Some backends expose job/pod deadlines. |
| `email_notifications` per task | Alerter stack component + hooks | approximate | ZenML alerters focus on chat (Slack/Discord). Email requires custom integration or alerter. Use `on_failure`/`on_success` hooks. |
| `webhook_notifications` per task | Custom hook calling webhook | approximate | Write a custom hook function that posts to the webhook URL. |
| `health` block notifications | Pipeline-level monitoring / alerters | approximate | Databricks health checks are per-task metrics-based. ZenML relies on step hooks and external monitoring. |

## Infrastructure and Compute

| Databricks Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Job clusters (`job_clusters[]` + `job_cluster_key`) | Orchestrator + step operator + `ResourceSettings` | approximate | Databricks clusters are Spark-centric with explicit lifecycle. ZenML compute is driven by orchestrator/step operator backend. Mapping is lossy unless target is also Databricks. |
| `existing_cluster_id` | Orchestrator's cluster config | approximate | In ZenML, the orchestrator manages compute. Using ZenML's Databricks orchestrator preserves existing cluster semantics. |
| Per-task `libraries[]` | `DockerSettings(requirements=..., apt_packages=...)` | approximate | Databricks installs libraries onto clusters at runtime. ZenML expects dependencies in the container image or per-step Docker settings. |
| Compute hints (`hardware_accelerator`) | `ResourceSettings(gpu_count=N)` | approximate | Databricks serverless GPU accelerators map to resource requests, but scheduling is backend-specific. |
| `new_cluster` node type / workers | `ResourceSettings(cpu_count=N, memory="XGi")` | approximate | Captures resource intent but not exact Spark cluster spec. |
| DBFS paths / mounts | Artifact store URIs or explicit cloud storage access | approximate | DBFS paths are Databricks-specific. In ZenML containers, use cloud storage URIs with proper credentials. **Flag as refactor hotspot.** |
| Unity Catalog tables/volumes | External data sources accessed via connectors | approximate | UC governance doesn't carry over. Treat UC paths as external sources with cloud storage credentials. |
| Databricks Repos / Git folders | ZenML code repositories | approximate | ZenML supports code repository registration. Move notebook/script assets into a proper importable repo. |
| `dbutils.secrets.get(scope, key)` | ZenML secrets store | direct/approximate | ZenML secrets can be injected into steps as environment variables. Map Databricks scope+key to ZenML secret name+key. |
| Databricks Connect (remote cluster) | Remote orchestration via orchestrator/step operator | approximate | ZenML runs steps remotely through orchestrators rather than client-side Spark sessions. |
| Job ACLs (`CAN_VIEW`, `CAN_MANAGE_RUN`) | ZenML Pro RBAC | approximate | ZenML OSS has limited access management; ZenML Pro provides RBAC across org/workspace/project scopes. |
| "Run as" user | Workload identity via stack + service connectors | approximate | Treat as "execution identity requirement" and flag for infrastructure mapping. |

## Stack Component Mappings

| Databricks Platform Feature | ZenML Stack Component | Mapping | Notes |
|---|---|:---:|---|
| Databricks-managed MLflow tracking | ZenML experiment tracker (MLflow integration) | direct/approximate | Configure with `tracking_uri="databricks"` and appropriate host/auth. |
| Databricks-managed MLflow Model Registry | ZenML model registry (MLflow integration) | approximate | Decide whether to keep MLflow registry or adopt ZenML Model Control Plane. |
| Delta Lake datasets | ZenML artifacts + materializers | approximate | Either keep Delta as storage format (access via connectors in steps) or materialize to other formats. |
| Databricks SQL warehouse | External warehouse accessed from steps | approximate | Write a step that runs SQL via Databricks SQL connector; store credentials via ZenML secrets. |
| Jobs UI | ZenML dashboard | approximate | Different visualization paradigm. Ensure important run info becomes explicit metadata/logs in ZenML. |
