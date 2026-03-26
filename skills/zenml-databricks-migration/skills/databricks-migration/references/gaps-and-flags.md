# Gaps, Flags, and Behavioral Differences

This reference covers the patterns that are most dangerous to silently approximate during migration -- the places where Databricks Workflows and ZenML behave fundamentally differently, and where a naive translation changes the pipeline's actual behavior.

## Table of Contents

- [Must-Flag Patterns](#must-flag-patterns)
- [Notebook Classification Guide](#notebook-classification-guide)
- [Behavioral Differences](#behavioral-differences)
- [Migration Decision Tree](#migration-decision-tree)
- [ZenML Features with No Databricks Equivalent](#zenml-features-with-no-databricks-equivalent)

---

## Must-Flag Patterns

These patterns must **never** be silently approximated. Flag them in the migration report and require human review.

### Non-default `run_if` conditions

**What Databricks does**: `run_if` controls when a task executes based on the status of upstream tasks. The default is `ALL_SUCCESS`. Other options: `ALL_DONE` (run regardless), `AT_LEAST_ONE_FAILED` (run only if some upstream failed), `NONE_FAILED` (run if no upstream failed, even if some were skipped).

**Why it matters**: `run_if` enables patterns like "always run cleanup regardless of success" or "send recovery notification if any upstream failed." Without them, the pipeline's error handling behavior changes silently.

**ZenML gap**: ZenML provides pipeline-level execution modes (`FAIL_FAST`, `STOP_ON_FAILURE`, `CONTINUE_ON_FAILURE`) but not per-step `run_if`. Steps run when their input artifacts are available (implying upstream success).

**Redesign approaches**:
- **`ALL_DONE` (cleanup steps)**: Use `execution_mode=CONTINUE_ON_FAILURE` at pipeline level + wrap upstream steps in try/except to return status artifacts. The cleanup step always executes because it receives a status artifact regardless of upstream success/failure.
- **`AT_LEAST_ONE_FAILED` (recovery/alerting)**: Use `on_failure` hooks on upstream steps instead of a downstream recovery task. Hooks run when the step fails, achieving the same notification effect.
- **`NONE_FAILED` (conditional continuation)**: The default ZenML behavior is similar (run if inputs available), but you lose the "skip" concept. Restructure so the step accepts optional inputs.

### File arrival triggers

**What Databricks does**: File arrival triggers monitor Unity Catalog volumes or external locations for new files, with configurable debounce (`wait_after_last_change_seconds`) and minimum interval (`min_time_between_triggers_seconds`). Tightly integrated with Unity Catalog permissions and managed file events.

**Why it matters**: This is a core pattern for incremental data ingestion pipelines. Removing it changes the pipeline from event-driven to manually-triggered or scheduled.

**ZenML gap**: ZenML OSS has no built-in file arrival trigger. No integration with Unity Catalog file events.

**Redesign approaches**:
- **Cloud-native eventing**: S3 Event Notifications, GCS Pub/Sub, Azure Event Grid -> message queue or webhook -> trigger ZenML pipeline run via deployment/snapshot API.
- **ZenML Pro triggers**: If using ZenML Pro, create a managed trigger for the pipeline.
- **Polling schedule**: As a simpler alternative, schedule the pipeline to run periodically and check for new files inside the first step (less efficient but simpler to implement).

### Continuous jobs

**What Databricks does**: Continuous jobs auto-restart runs after completion, with optional `pause_status` control. Designed for always-on streaming workloads.

**Why it matters**: ZenML pipelines are batch-oriented (run to completion). An always-on streaming pattern is fundamentally different.

**ZenML gap**: No native "continuous job" primitive. ZenML has pipeline deployments for HTTP-triggered execution, but not auto-restarting execution.

**Redesign approaches**:
- **Recurring schedule**: For near-real-time, use a tight cron schedule (e.g., every 5 minutes).
- **External orchestration**: Keep the streaming component as a separate long-running service (e.g., Spark Structured Streaming on Kubernetes) and use ZenML for the batch/ML portions.
- **Pipeline deployment + webhook**: Set up a webhook that triggers the pipeline on events.

### Notebooks with platform-coupled patterns

**What Databricks does**: Notebooks use `%run` to import other notebooks, `%pip` for runtime library installs, `%sql` for inline SQL execution, `display()` for rich output, and `dbutils.fs` for workspace filesystem access. These are all notebook kernel features.

**Why it matters**: ZenML steps run as Python functions in containers, not in notebook kernels. None of these features are available.

**ZenML gap**: No notebook runtime. Steps must be pure Python callables with explicit dependencies and typed I/O.

**When to flag** (always flag as HIGH severity):
- `%run` importing other notebooks (implicit code loading)
- `%pip` installing libraries at runtime (not reproducible in containers)
- `%sql` executing SQL as a magic (not valid Python)
- `%sh` executing shell commands for critical operations
- `display()` used for decision-making (not just visualization)
- Mixed-language cells (Python + SQL + Scala)

### Shared cluster state

**What Databricks does**: Tasks sharing a `job_cluster_key` or `existing_cluster_id` can share Spark context, cached tables, temporary views, and driver-local files. The warm cluster preserves state between tasks.

**Why it matters**: Hidden data dependencies not represented in the job DAG. Downstream tasks may assume temp views or cached data exist from upstream tasks.

**ZenML gap**: Each step runs in its own container (default). No shared Spark context, no `/tmp` handoffs, no cached tables.

**Redesign approaches**:
- **Make data dependencies explicit**: Upstream steps must return data as artifacts; downstream steps read those artifacts as inputs.
- **Use a shared data store**: If tasks communicated via Delta tables, continue using them as the data layer (steps read/write via connectors).
- **Databricks orchestrator**: If keeping Databricks as the execution backend, ZenML's Databricks orchestrator runs steps as Databricks tasks, preserving some cluster semantics.

### DBFS paths and workspace paths

**What Databricks does**: Tasks use DBFS paths (`dbfs:/...`) and workspace paths (`/Workspace/...`, `/Repos/...`) that only exist in the Databricks environment.

**Why it matters**: These paths don't exist in containers running on other orchestrators. Passing them between tasks silently breaks.

**Redesign approaches**:
- Replace DBFS paths with cloud storage URIs (`s3://`, `gs://`, `abfss://`)
- Use ZenML's artifact store for step-to-step data passing
- Mount cloud storage in containers if path-based access is required

### SQL/dbt tasks with managed identity

**What Databricks does**: `sql_task` and `dbt_task` automatically receive Databricks-managed authentication tokens. No explicit credential configuration needed.

**Why it matters**: Outside Databricks, there are no managed tokens. The step needs explicit credentials.

**Redesign approaches**:
- Store credentials in ZenML secrets (`zenml secret create`)
- Use ZenML service connectors for cloud authentication
- Configure dbt profiles with explicit connection details

---

## Notebook Classification Guide

Every notebook task should be classified before migration. Use this checklist:

### Detection checklist

| Pattern | Detection | Risk Level |
|---|---|---|
| `dbutils.widgets.get/text/dropdown` | Widget parameter access | LOW -- maps to step params |
| `dbutils.jobs.taskValues.set/get` | Task value data passing | LOW -- maps to artifacts |
| `dbutils.secrets.get` | Secret access | LOW -- maps to ZenML secrets |
| `dbutils.fs.*` | DBFS filesystem operations | MEDIUM -- refactor to cloud storage |
| `spark.read.table()` / `spark.sql()` | Spark session usage | MEDIUM -- needs data access decision |
| `createOrReplaceTempView()` | Shared Spark state | HIGH -- implicit data dependency |
| `%sql` | SQL magic | HIGH -- not valid Python |
| `%pip install` | Runtime library install | HIGH -- use DockerSettings |
| `%run /path/to/notebook` | Notebook import | HIGH -- refactor to Python modules |
| `%sh` | Shell command magic | MEDIUM -- use subprocess.run() |
| `display()` | Rich visualization | LOW -- replace with log_metadata() |
| Mixed language cells | Python + SQL + Scala | HIGH -- requires full manual refactor |

### Classification outcomes

Based on detected patterns, classify each notebook into one of:

**Auto-refactorable** (low risk):
- Uses `dbutils.widgets` for params and `dbutils.jobs.taskValues` for output
- Otherwise standard Python/pandas code
- No magics, no shared Spark state, no DBFS paths
- Action: Extract logic into `@step` function with typed parameters

**Semi-automatic** (medium risk):
- Uses `spark.read.table()` but no temp views shared across tasks
- Uses `dbutils.fs` but only for local operations within the notebook
- Uses `%sh` for non-critical operations
- Action: Extract logic, decide on data access strategy (Spark vs pandas), flag Spark dependency

**Manual refactor required** (high risk):
- Uses `%run`, `%pip`, `%sql` magics
- Creates temp views consumed by downstream notebook tasks
- Mixes languages (Python + SQL cells driving control flow)
- Relies on DBFS mounts for inter-task data sharing
- Action: Flag as HIGH severity, provide refactoring guidance, require human review

---

## Behavioral Differences

### Task values vs Artifacts

| Aspect | Databricks Task Values | ZenML Artifacts |
|---|---|---|
| **Size limit** | 48 KiB JSON | Unlimited (stored in artifact store) |
| **Type system** | JSON-only (string, number, bool, array, object) | Python types with materializer-based serialization |
| **Persistence** | Ephemeral per run | Versioned, persisted across runs |
| **Access pattern** | `dbutils.jobs.taskValues.get()` or `{{tasks.*.values.*}}` | Function parameter in step signature |
| **Caching** | Not cached | ZenML can skip re-execution when inputs haven't changed |
| **Lineage** | Not tracked | Full artifact lineage graph |

### Execution model

| Aspect | Databricks Workflows | ZenML Pipelines |
|---|---|---|
| **Task isolation** | Shared or per-task Spark clusters | One container per step (default) |
| **State sharing** | Possible via shared cluster (temp views, caches) | Not possible -- artifacts only |
| **Compute lifecycle** | Explicit cluster management (create/reuse/terminate) | Orchestrator-managed containers |
| **Warm state** | Spark context persists on shared clusters | Each step starts fresh |
| **File system** | DBFS, workspace paths, cluster-local /tmp | Container-local filesystem, cloud storage URIs |
| **Language support** | Python, SQL, Scala, R (in notebooks) | Python only |
| **Runtime environment** | Notebook kernel with dbutils, magics, display() | Pure Python callable with explicit I/O |

### Parameterization

| Aspect | Databricks | ZenML |
|---|---|---|
| **Mechanism** | String substitution (`{{...}}`) | Typed Python function parameters |
| **Error handling** | Some syntax errors silently treated as literals | Standard Python type errors |
| **Type safety** | All values are strings after substitution | Full Python type system |
| **Sources** | Job parameters, dynamic refs, widgets, named_parameters | Pipeline args, YAML config, step parameters |
| **Scope** | Multiple channels (job-level, task-level, notebook-level) | Unified Python parameter model |

### Scheduling and triggers

| Aspect | Databricks | ZenML |
|---|---|---|
| **Cron** | 6-field Quartz cron with timezone | 5-field standard cron, orchestrator-dependent |
| **Interval** | Periodic trigger with explicit unit | `Schedule(interval_second=N)` |
| **File arrival** | Native Unity Catalog integration | Not available (external eventing needed) |
| **Table update** | Native trigger type | Not available |
| **Continuous** | Auto-restart on completion | Not available (use recurring schedule) |
| **Lifecycle management** | Databricks manages trigger state | Orchestrator-dependent (limited in OSS) |

---

## Migration Decision Tree

Text-based decision procedure for translating Databricks tasks to ZenML steps.

```
INPUT: task_type, task_config, notebook_patterns (if notebook_task)

TASK TYPE MAPPING:
  IF task_type == "notebook_task":
    IF notebook has %run/%pip/%sql magics OR relies on shared temp views:
      FLAG blocker NOTEBOOK_REFACTOR_REQUIRED
      RECOMMEND: manual refactor into Python modules with explicit I/O
    ELSE IF notebook uses Spark heavily (read.table, DataFrame ops):
      FLAG warn SPARK_DEPENDENCY
      RECOMMEND: decide data access strategy (keep Spark via step operator OR refactor to pandas)
      EMIT @step wrapping extracted functions
    ELSE:
      EMIT @step with widget params -> function params, taskValues -> return values

  ELSE IF task_type IN {"spark_python_task", "python_wheel_task"}:
    IF entrypoint is importable Python function:
      EMIT @step calling that function with DockerSettings for dependencies
    ELSE:
      EMIT @step that runs the entry point in a container
      FLAG warn ENTRY_POINT_WRAPPED

  ELSE IF task_type IN {"sql_task", "dbt_task"}:
    EMIT @step executing SQL/dbt via explicit client
    REQUIRE secrets: credentials in ZenML secret store
    FLAG warn EXTERNAL_SYSTEM_DEPENDENCY

  ELSE IF task_type IN {"spark_jar_task", "spark_submit_task"}:
    IF target stack supports Spark step operator:
      EMIT step with step operator settings
    ELSE:
      FLAG blocker SPARK_BACKEND_UNDEFINED
      RECOMMEND: choose Spark backend (Databricks orchestrator, K8s Spark, etc.)

  ELSE IF task_type == "condition_task":
    IF condition depends only on pipeline parameters:
      EMIT static pipeline if/else
    ELSE:
      EMIT @pipeline(dynamic=True) with .load() for runtime branching
      FLAG warn DYNAMIC_PIPELINE_REQUIRED

  ELSE IF task_type == "for_each_task":
    EMIT @pipeline(dynamic=True) with .map() for fan-out
    FLAG warn DYNAMIC_PIPELINE_REQUIRED
    NOTE: original concurrency setting is advisory-only in ZenML

  ELSE IF task_type == "run_job_task":
    IF target job exists in same codebase:
      EMIT pipeline composition (call pipeline function directly)
    ELSE:
      EMIT step that triggers pipeline via ZenML Client API
      FLAG warn CROSS_WORKSPACE_DEPENDENCY

CONTROL FLOW:
  IF task has run_if != "ALL_SUCCESS":
    IF run_if == "ALL_DONE":
      RECOMMEND: execution_mode=CONTINUE_ON_FAILURE + status artifacts
    ELSE IF run_if == "AT_LEAST_ONE_FAILED":
      RECOMMEND: on_failure hooks on upstream steps
    FLAG blocker RUN_IF_REQUIRES_REDESIGN

DATA PASSING:
  IF task uses dbutils.jobs.taskValues or {{tasks.*.values.*}}:
    MAP to typed artifacts (step return values -> step input parameters)
  IF task uses {{job.parameters.*}} or dbutils.widgets:
    MAP to pipeline/step function parameters with explicit types

COMPUTE:
  IF task references job_cluster_key or existing_cluster_id:
    IF target stack is Databricks (ZenML Databricks orchestrator):
      Preserve cluster config in orchestrator settings
    ELSE:
      Translate to ResourceSettings + step operator settings
      FLAG warn COMPUTE_SEMANTICS_CHANGED

RETRIES/TIMEOUTS:
  IF max_retries > 0:
    EMIT @step(retry=StepRetryConfig(...))
  IF timeout_seconds > 0:
    IF target orchestrator supports hard timeouts:
      Configure orchestrator/job deadline settings
    ELSE:
      FLAG warn TIMEOUT_ENFORCEMENT_APPROXIMATE

TRIGGERS:
  IF job has file_arrival or table_update or continuous trigger:
    FLAG blocker EVENT_TRIGGER_REQUIRES_INFRA_DECISION
    RECOMMEND: external eventing + webhook + pipeline deployments
  ELSE IF job has cron schedule:
    IF target orchestrator supports scheduling:
      EMIT Schedule(cron_expression=...) (convert 6-field Quartz to 5-field)
    ELSE:
      FLAG warn SCHEDULE_MANAGEMENT_LIMITED
```

---

## ZenML Features with No Databricks Equivalent

These are capabilities the user gains after migration -- include relevant ones in the "What You Get for Free" section of the migration report.

| ZenML Feature | Description |
|---|---|
| **First-class typed artifacts** | Full datasets, models, images -- not just 48KiB JSON blobs. Versioned, persisted, and traceable. |
| **Step caching** | Skip re-execution when code and inputs haven't changed. Databricks has no equivalent. |
| **Stack abstraction** | Same pipeline code runs on local, Kubernetes, Vertex AI, SageMaker, AzureML by switching stacks. Databricks locks you to one platform. |
| **Model Control Plane** | Track ML models with versioning, promotion stages, and lineage. Goes beyond MLflow model registry. |
| **Service connectors** | Unified cloud authentication with automatic token refresh. Replaces Databricks-specific secret scopes and managed identity. |
| **Pipeline execution modes** | `FAIL_FAST`, `STOP_ON_FAILURE`, `CONTINUE_ON_FAILURE` control failure behavior at the pipeline level. |
| **Step/pipeline hooks** | `on_failure`, `on_success` hooks for cross-cutting concerns without dedicated notification tasks. |
| **Dynamic pipelines** | Runtime-shaped DAGs with `.map()` and `.load()`. More expressive than `for_each_task` for complex patterns. |
| **Alerter components** | Slack/Discord alerters and human-in-the-loop ask steps, beyond email/webhook notifications. |
| **Code repository integration** | First-class concept for reproducibility and remote execution tracking. |
| **Artifact lineage graph** | Full provenance tracking across pipeline runs, not just within a single job. |
