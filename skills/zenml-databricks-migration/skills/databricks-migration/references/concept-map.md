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
| `python_wheel_task` | `@step` calling the wheel entry point as a Python function | approximate | Databricks runs an installed wheel using `package_name` + `entry_point`. In ZenML, import and call internal functions directly; use `DockerSettings`/`pyproject.toml` for dependencies. DBFS/workspace wheels need a private index, direct URL, or Docker build artifact strategy. |
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
| `run_if: ALL_DONE` | Pipeline execution mode + explicit failure handling | absent | Static ZenML pipelines have pipeline-level `execution_mode=CONTINUE_ON_FAILURE`, but ZenML has no per-step `run_if`. Dynamic pipelines currently do not support `CONTINUE_ON_FAILURE`, so `for_each_task`/runtime-branching migrations need explicit status artifacts or a different failure-domain design. **Always flag.** |
| `run_if: AT_LEAST_ONE_FAILED` | `on_failure` hooks or status-checking step | absent | No direct equivalent. Use hooks on upstream steps instead of a downstream recovery task. **Always flag.** |
| `run_if: NONE_FAILED` | Approximate via default behavior | approximate | Default ZenML behavior is similar (run if inputs available), but semantics differ when some upstream steps are skipped. |
| Cron scheduling (`quartz_cron_expression`) | `Schedule(cron_expression=...)` + optional orchestrator settings | approximate | Databricks uses Quartz cron with timezone; ZenML uses standard 5-field cron. Convert simple expressions by dropping a zero seconds field, but flag non-zero seconds, year fields, or Quartz-only operators (`?`, `L`, `W`, `#`) for human review. If the target is the Databricks orchestrator, set `DatabricksOrchestratorSettings(schedule_timezone="<IANA timezone>")` with examples like `UTC`, `America/New_York`, or `America/Los_Angeles`. |
| Periodic trigger (`trigger.periodic`) | `Schedule(interval_second=...)` or ZenML Pro schedule trigger | approximate | Databricks periodic trigger has explicit interval + unit. ZenML supports interval seconds on some orchestrators, but the Databricks orchestrator only supports cron schedules and ignores interval fields. |
| File arrival trigger | External eventing + deployment/snapshot trigger; possibly ZenML Pro triggers for supported ZenML platform events | absent | Databricks integrates with Unity Catalog file events. ZenML OSS has no native file arrival trigger, and current ZenML Pro platform-event triggers listen to ZenML platform events rather than arbitrary Unity Catalog file arrivals. Requires Databricks/cloud events -> webhook/service -> pipeline run. **Always flag.** |
| Table update trigger | External eventing + deployment/snapshot trigger; possibly ZenML Pro triggers for supported ZenML platform events | absent | Similar to file arrival. Do not imply 1:1 parity with current ZenML Pro platform-event triggers unless the source event is a supported ZenML platform event. **Always flag.** |
| Continuous jobs | Separate streaming service, recurring schedule, or HTTP pipeline deployment depending on intent | absent | Databricks continuous mode auto-restarts always-on workloads. ZenML pipelines are run-to-completion; deployments are HTTP services; `zenml.streaming.publish()` is live telemetry from a run, not streaming compute. **Always flag.** |

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
| `{{job.parameters.*}}` | Pipeline function parameters populated from YAML config | approximate | Databricks replaces with string literal; ZenML parameters are typed Python values. Must parse and type-cast. Put business values and environment-specific settings in populated YAML configs, not a long `argparse` surface. |
| `dbutils.widgets.get()` / `dbutils.widgets.text()` | Step function parameters populated from pipeline config | approximate | Widgets are notebook-runtime API. In ZenML, the step signature defines parameters; values come from pipeline call or YAML config. |
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
| Whole Databricks job runtime | ZenML Databricks orchestrator + `DatabricksOrchestratorSettings` | approximate | Use when Databricks should run the full pipeline as a persistent Databricks Job. Job-level settings include `job_tags`, `max_concurrent_runs`, `max_retries`, `min_retry_interval_millis`, `retry_on_timeout`, `timeout_seconds`, and `schedule_timezone` for cron schedules. |
| Selected Databricks task runtime | ZenML Databricks step operator + `DatabricksStepOperatorSettings` | approximate | Use when another orchestrator owns the pipeline but selected steps run on Databricks as one-time `jobs.submit` runs. The step operator does not own schedules and does not expose persistent-job settings such as `schedule_timezone`, `job_tags`, `max_concurrent_runs`, `max_retries`, `min_retry_interval_millis`, or `retry_on_timeout`. |
| Job clusters (`job_clusters[]` + `job_cluster_key`) | Databricks-specific orchestrator/step-operator settings | approximate | Databricks clusters are Spark-centric with explicit lifecycle. For Databricks targets, configure `spark_version`, `num_workers` or `autoscale`, `node_type_id`, `driver_node_type_id`, `policy_id`, `spark_conf`, `spark_env_vars`, `custom_tags`, and related Databricks settings. Generic `ResourceSettings` does not size Databricks step-operator clusters. |
| `existing_cluster_id` | Databricks orchestrator/step-operator cluster config or redesign | approximate | In ZenML, the selected stack component manages compute. Preserve Databricks cluster semantics only when the target runtime remains Databricks. |
| Per-task `libraries[]` | `DockerSettings(requirements=..., apt_packages=...)`, `pyproject.toml`, or private package artifact | approximate | Databricks installs libraries onto clusters at runtime. ZenML expects dependencies in the container image or per-step Docker settings. Inventory `whl`, `pypi`, `maven`, `jar`, and workspace/DBFS paths; unresolved DBFS/workspace wheels are migration blockers, not vague TODOs. |
| Compute hints (`hardware_accelerator`) | Backend-specific resource config | approximate | For non-Databricks orchestrators, `ResourceSettings(gpu_count=N)` may capture resource intent. For Databricks step-operator or orchestrator execution, use GPU-capable Databricks Spark versions and node types in Databricks-specific settings. |
| `new_cluster` node type / workers | `DatabricksOrchestratorSettings` or `DatabricksStepOperatorSettings` | approximate | Map node type, fixed workers, autoscaling bounds, Spark config, init scripts, and Docker image settings to Databricks-specific settings. Do not rely on generic `ResourceSettings` to create a Databricks cluster of the right size. |
| DBFS paths / mounts | Artifact store URIs or explicit cloud storage access | approximate | DBFS paths are Databricks-specific. In ZenML containers, use cloud storage URIs with proper credentials. **Flag as refactor hotspot.** |
| Unity Catalog tables/volumes | External data sources accessed via connectors | approximate | UC governance doesn't carry over. Treat UC paths as external sources with cloud storage credentials. |
| Databricks Feature Engineering Client / UC feature lookup | Databricks-native feature access or explicit feature-store redesign | approximate/high-caveat | Preserve point-in-time lookup semantics, feature metadata, online/offline behavior, and UC permissions. Do not rewrite to ordinary SQL unless it is truly a simple table read and the user accepts the semantic change. |
| Databricks Repos / Git folders | ZenML code repositories | approximate | ZenML supports code repository registration. Move notebook/script assets into a proper importable repo. |
| `dbutils.secrets.get(scope, key)` | ZenML secrets store | direct/approximate | ZenML secrets can be injected into steps as environment variables. Map Databricks scope+key to ZenML secret name+key. |
| Databricks Connect (remote cluster) | Remote orchestration via Databricks orchestrator or step operator | approximate | Use the orchestrator for whole-pipeline Databricks execution; use the step operator for selected Databricks steps under another orchestrator. ZenML runs packaged step entrypoints remotely rather than relying on a client-side Spark session. |
| Job ACLs (`CAN_VIEW`, `CAN_MANAGE_RUN`) | ZenML Pro RBAC | approximate | ZenML OSS has limited access management; ZenML Pro provides RBAC across org/workspace/project scopes. |
| "Run as" user | Workload identity via stack + service connectors | approximate | Treat as "execution identity requirement" and flag for infrastructure mapping. |

## Stack Component Mappings

| Databricks Platform Feature | ZenML Stack Component | Mapping | Notes |
|---|---|:---:|---|
| Databricks-managed MLflow tracking | ZenML experiment tracker (MLflow integration) | direct/approximate | Configure with `tracking_uri="databricks"` and appropriate host/auth. |
| Databricks-managed MLflow Model Registry | ZenML model registry (MLflow integration) | approximate | Decide whether to keep MLflow registry or adopt ZenML Model Control Plane. |
| Delta Lake datasets | ZenML artifacts + materializers | approximate | Either keep Delta as storage format (access via connectors in steps) or materialize to other formats. |
| Databricks SQL warehouse | External warehouse accessed from steps | approximate | Write a step that runs SQL via Databricks SQL connector; store credentials via ZenML secrets. Do not use this as the default replacement for Feature Engineering Client point-in-time lookups. |
| Jobs UI | ZenML dashboard + Databricks run/job URLs | approximate | Different visualization paradigm. Runs executed on Databricks still have Databricks UI records; ensure important run info becomes explicit metadata/logs in ZenML. |
