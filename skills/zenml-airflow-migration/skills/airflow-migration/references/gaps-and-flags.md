# Gaps, Flags, and Behavioral Differences

This reference covers the patterns that are most dangerous to silently approximate during migration — the places where Airflow and ZenML behave fundamentally differently, and where a naive translation changes the pipeline's actual behavior.

## Table of Contents

- [Must-Flag Patterns](#must-flag-patterns)
- [Behavioral Differences](#behavioral-differences)
- [Migration Decision Tree](#migration-decision-tree)
- [ZenML Features with No Airflow Equivalent](#zenml-features-with-no-airflow-equivalent)

---

## Must-Flag Patterns

These patterns must **never** be silently approximated. Flag them in the migration report and require human review.

### Non-default trigger rules

**What Airflow does**: Trigger rules control when a task runs based on the status of its upstream tasks. The default is `all_success` (run only if all upstreams succeeded). Other rules include `all_done` (run regardless), `one_failed` (run if any upstream failed), `none_failed_min_one_success`, etc. These are fundamental to Airflow's state machine.

**Why it matters**: Trigger rules enable patterns like "always run cleanup regardless of success" or "send alert if any upstream failed." Without them, the pipeline's error handling behavior changes silently.

**ZenML gap**: No equivalent trigger-rule system. Steps run when their input artifacts are available (implying upstream success).

**Redesign approaches**:
- **`all_done` (cleanup steps)**: Wrap upstream steps in try/except, return a status artifact, and have the cleanup step always execute based on that status. Or split into separate pipelines with independent failure domains.
- **`one_failed` (alerting)**: Use `on_failure` hooks on upstream steps instead of a downstream alert task.
- **Join-after-branch** (`none_failed_min_one_success`): Restructure so both branches produce an artifact and the join step accepts optional inputs.

### Branching on upstream task outputs

**What Airflow does**: `BranchPythonOperator` can read XCom values from upstream tasks and decide which downstream branch to execute. Non-selected branches get "skipped" status.

**Why it matters**: This is runtime conditional execution driven by data computed during the pipeline run.

**ZenML gap**: ZenML's pipeline graph is constructed before execution (for static pipelines). Branching based on pipeline parameters works fine, but branching based on step output values requires a different approach.

**Redesign approaches**:
- **Dynamic pipeline**: Use `@pipeline(dynamic=True)`, call `.load()` on the upstream output, and conditionally invoke steps. Limited orchestrator support.
- **Both-branches-run pattern**: Execute both branches but have each check a condition internally and no-op when the condition doesn't apply. Simple but wastes compute.
- **Separate pipelines**: Split the pre-branch and post-branch parts into separate pipelines, with the branching logic in the triggering code (e.g., `run.py`).

### Dynamic task mapping with runtime cardinality

**What Airflow does**: `expand()` (Airflow 2.3+) creates mapped task instances at runtime. The number of tasks is determined by an upstream task's output. Each mapped instance has independent retry, logging, and observability.

**Why it matters**: This is a core Airflow pattern for processing variable-length collections in parallel with per-item observability.

**ZenML gap**: ZenML explicitly does not support "dynamically creating steps on the fly" from upstream outputs in static pipelines. Dynamic pipelines (`@pipeline(dynamic=True)`) support `.map()` but with limited orchestrator support and different semantics.

**Redesign approaches**:
- **Dynamic pipeline with `.map()`**: The closest equivalent. Requires `@pipeline(dynamic=True)` and a supported orchestrator (Local, LocalDocker, Kubernetes, Vertex, SageMaker, AzureML).
- **Static fan-out**: If the list can be made a pipeline parameter (known before execution), use a loop with explicit step IDs.
- **Multi-run pattern**: Each item becomes a separate pipeline run. Provides the strongest isolation and retry semantics but changes the architecture fundamentally.
- **Single-step loop**: Process all items in one step. Simplest but loses per-item observability and retry.

### Sensors and deferrable operators

**What Airflow does**: Sensors wait for external conditions with dedicated scheduling semantics. `mode="poke"` holds a worker slot. `mode="reschedule"` releases the slot between checks. Deferrable operators offload waiting to a lightweight triggerer process.

**Why it matters**: Sensors are how Airflow handles external dependencies (file arrival, API readiness, time-based waits) without burning compute.

**ZenML gap**: No sensor primitive, no deferral mechanism, no triggerer process. A polling step in ZenML consumes a full compute slot (container/pod) for its entire duration.

**Redesign approaches**:
- **Orchestrator scheduling**: Instead of a sensor waiting for data, schedule the pipeline to run after the expected arrival time. Simpler and more resource-efficient.
- **External event trigger**: Use webhooks or external systems to trigger pipeline runs when the condition is met.
- **Polling step with timeout**: If polling is unavoidable, implement it as a step with aggressive timeout and reasonable poll intervals. Note the resource cost in the migration report.

### Pools and priority weights for correctness

**What Airflow does**: Pools limit concurrency across tasks (e.g., "only 2 tasks can hit this API simultaneously"). Priority weights influence execution order in the queue.

**Why it matters**: When pools are used for rate limiting (preventing API throttling, respecting external system limits), removing them changes correctness, not just performance.

**ZenML gap**: No pool or priority-weight API. Concurrency is managed by the orchestrator infrastructure.

**Redesign approaches**:
- **Rate limiting in step code**: Add explicit rate limiting (e.g., `time.sleep()` between API calls, or a rate limiter library) inside the step.
- **Orchestrator concurrency settings**: Some orchestrators expose concurrency limits at the infrastructure level.
- **Sequential execution**: If the pool allowed only 1 concurrent task, ensure the steps are wired sequentially in the pipeline.

### SubDagOperator-based control flow

**What Airflow does**: SubDAGs embed a complete DAG within a task. Deprecated since Airflow 2.0 in favor of TaskGroups, but still present in older codebases.

**Why it matters**: SubDAGs have complex lifecycle semantics (their own scheduler interaction, deadlock potential).

**Redesign**: Always refactor SubDAGs into either TaskGroup-equivalent composition functions (if the logic is simple) or separate pipelines (if the SubDAG represents an independent unit of work). Do not attempt to preserve SubDAG semantics.

---

## Behavioral Differences

These are the semantic differences that are safe to translate but important to communicate to the user — the "it works but means something different" cases.

### XCom vs artifacts

| Aspect | Airflow XCom | ZenML Artifacts |
|---|---|---|
| **Storage** | Metadata DB (by default) | Artifact store (S3, GCS, local, etc.) |
| **Size** | Designed for small values; custom backends for large | Arbitrarily large (DataFrames, models, images) |
| **Serialization** | JSON/pickle (configurable) | Materializers (type-specific, explicit) |
| **Lifecycle** | Ephemeral per DAG run (cleaned up with run) | Versioned, persisted across runs |
| **Caching** | None | Steps can be skipped when inputs unchanged |
| **Mutability** | Mutable (can overwrite XCom keys) | Immutable after creation |
| **Lineage** | Limited (which task produced it) | Full artifact lineage graph |

**Practical impact**: Migration may need to handle serialization differences. If Airflow tasks pass large objects via custom XCom backends, the ZenML materializer must handle the same data efficiently. Cloudpickle serialization in XCom should be replaced with explicit materializers in ZenML.

### Execution model

| Aspect | Airflow | ZenML |
|---|---|---|
| **Scheduling** | Central scheduler with state machine | Delegated to orchestrator |
| **Task isolation** | Worker process/pod (depends on executor) | Container/pod per step (on remote orchestrators) |
| **Shared state** | Metadata DB, XCom, Airflow context | Artifact store, ZenML server |
| **Filesystem** | Workers may share filesystem (Local/Celery executor) | No shared filesystem on remote orchestrators |
| **Context** | Rich context dict (`ti`, `dag_run`, `ds`, `execution_date`) | Step context (limited), pipeline context |
| **Failure handling** | Task instance state machine (up_for_retry, failed, skipped) | Step status (running, completed, failed) |

**Practical impact**: Any Airflow code that accesses `context["ti"]`, `context["dag_run"]`, `context["ds"]`, or similar Airflow-specific context must be rewritten. ZenML provides `get_step_context()` for limited step metadata.

### Scheduling semantics

| Aspect | Airflow | ZenML |
|---|---|---|
| **Schedule ownership** | Airflow scheduler (core component) | Delegated to orchestrator |
| **Data intervals** | First-class concept (logical date, data interval) | No equivalent |
| **Catchup/backfill** | Built-in (creates runs for missed intervals) | Orchestrator-dependent |
| **Timetables** | Custom timetable API (Airflow 2.2+) | No equivalent |
| **Schedule management** | Airflow CLI/UI | `zenml pipeline schedule` CLI (limited orchestrator support) |

**Practical impact**: If the Airflow DAG relies on data interval semantics (e.g., `{{ ds }}` to determine which date's data to process), this logic must be explicitly implemented in the ZenML pipeline (e.g., as a pipeline parameter for the target date).

### Retry semantics

| Aspect | Airflow | ZenML |
|---|---|---|
| **Retry scope** | Task instance level | Step level |
| **Backoff** | Boolean (`retry_exponential_backoff`) | Numeric factor (`backoff=2`) |
| **Max retry delay** | Configurable (`max_retry_delay`) | Not configurable |
| **Orchestrator retries** | Separate from task retries | May overlap (e.g., K8s pod restart + step retry) |
| **Idempotency** | User's responsibility | User's responsibility |

**Practical impact**: On Kubernetes, pod failures may trigger restarts independently of ZenML step retries. A step configured with `max_retries=3` that runs on a Kubernetes orchestrator with pod restart policies may retry more than 3 times total. Document this for non-idempotent operations.

---

## Migration Decision Tree

Use this to systematically classify each Airflow task for migration:

```
1) Classify task type
   ├── PythonOperator / @task → "python_step" → translate directly
   ├── BashOperator → "subprocess_step" → translate with subprocess.run()
   ├── Sensor / BaseSensorOperator → "sensor" → FLAG, redesign needed
   ├── KubernetesPodOperator-like → "container_job" → translate with step operator
   ├── SparkSubmitOperator-like → "spark_job" → translate with Spark step operator
   └── Custom/provider operator → "custom" → analyze intent, translate body

2) Classify data flow
   ├── Task pushes/pulls XCom for data → "data_passing" → translate to artifacts
   ├── XCom used for control flow (branching, mapping) → FLAG HIGH: "control_plane_xcom"
   └── Task reads/writes external storage → "external_io" → first step loads, rest use artifacts

3) Check control flow
   ├── trigger_rule != all_success → FLAG HIGH: "unsupported_trigger_rule"
   ├── Dynamic task mapping with expand() → FLAG HIGH: "dynamic_mapping"
   │   ├── Cardinality from pipeline parameter → static fan-out possible
   │   └── Cardinality from upstream output → dynamic pipeline or multi-run
   └── Branching depends on upstream outputs → FLAG HIGH: "runtime_branching"

4) Map infrastructure
   ├── Connections/variables → secrets + stack components + service connectors
   ├── K8s cluster required → Kubernetes orchestrator or step operator
   ├── Cloud resources → cloud artifact store + service connector
   └── Pools for rate limiting → in-step rate limiting

5) Decide: auto-migrate vs flag
   ├── Any HIGH flags? → Produce partial translation + flags + redesign suggestions
   └── No HIGH flags? → Full auto-migration with approximate-translation notes
```

---

## ZenML Features with No Airflow Equivalent

These are capabilities the user gains after migration. Mention relevant ones in the migration report as "what you get for free":

| Feature | Value |
|---|---|
| **Artifact versioning and lineage** | Every step output is versioned and linked to the code and inputs that produced it. Enables reproducibility and comparison across runs. |
| **Step caching** | Steps can be skipped when code and inputs haven't changed, saving compute on reruns. Must be intentionally enabled/disabled. |
| **Stack abstraction** | The same pipeline code runs on different infrastructure (local, K8s, Vertex, SageMaker) by switching stacks — no code changes. |
| **Model Control Plane** | Track ML models as first-class entities with versioning, promotion stages (staging → production), and cross-pipeline artifact sharing. |
| **Service connectors** | Unified authentication abstraction for cloud services, with automatic token refresh. Replaces ad-hoc connection/secret management. |
| **First-class MLOps components** | Experiment trackers, model registries, data validators, and deployers are stack components — not tasks you hand-wire. |
| **Pipeline deployment (HTTP serving)** | Wrap a pipeline as an HTTP service with `pipeline.deploy()` for real-time inference or agent workflows. |
| **Metadata logging** | Log metrics, parameters, and results from within steps — visible in the dashboard. |
