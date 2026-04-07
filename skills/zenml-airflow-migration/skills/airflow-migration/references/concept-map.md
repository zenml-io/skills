# Airflow → ZenML Concept Map

Complete mapping of Airflow concepts to their ZenML equivalents. Each entry is classified as **direct** (clean 1:1 mapping), **approximate** (conceptual equivalent with different semantics), or **absent** (no ZenML equivalent — needs redesign).

## Table of Contents

- [Core Concepts](#core-concepts)
- [Control Flow and Scheduling](#control-flow-and-scheduling)
- [Data Passing and Parameters](#data-passing-and-parameters)
- [Error Handling and Monitoring](#error-handling-and-monitoring)
- [Infrastructure and Operations](#infrastructure-and-operations)
- [Stack Component Mappings](#stack-component-mappings)

## Core Concepts

| Airflow Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| DAG | `@pipeline` | direct | Both define a directed acyclic graph of execution. ZenML pipelines are Python functions decorated with `@pipeline`; steps are invoked by calling step objects inside the function. |
| Operator (general) | `@step` or `BaseStep` subclass | direct | Airflow operators are task nodes; ZenML steps are nodes producing/consuming artifacts. |
| `PythonOperator` | `@step` running Python code | direct | Move the callable body into a `@step` function. No Airflow context dict available in ZenML. |
| TaskFlow API `@task` | `@step` | direct | Both wrap Python functions and treat returns as outputs. Airflow returns become XComs; ZenML returns become persisted artifacts. |
| Classic operator instantiation (`task = Operator(...)`) | Step decorator/function definitions | direct | Operator args → step function arguments + step/pipeline configuration. |
| Task dependencies (`>>`, `<<`, `set_upstream`) | Implicit via step invocation order and data passing | direct (common), approximate (ordering-only deps) | ZenML dependencies are induced by passing outputs to inputs. Pure ordering dependencies (no data flow) require passing a dummy dependency or using orchestrator patterns. |
| `BashOperator` | `@step` with `subprocess.run()` | approximate | No "BashOperator" primitive in ZenML. Containerization and working directory assumptions differ on remote stacks. |
| `BranchPythonOperator` / `@task.branch` | Conditional logic in pipeline function | approximate | ZenML can branch on **pipeline parameters** (values known at construction time). Cannot branch on upstream **step outputs** without redesign. |
| `ShortCircuitOperator` / `@task.short_circuit` | Guard step or parameter-driven conditional | approximate | Airflow short-circuit skips downstream tasks. ZenML uses explicit conditions or no-op steps. No "skipped state" semantics. |
| TaskGroups (`TaskGroup`, `@task_group`) | Python composition functions | approximate | Airflow TaskGroups are UI grouping + default-arg scoping. ZenML achieves modularity via plain Python functions that wire steps. No UI grouping equivalent. |
| SubDAGs (`SubDagOperator`) | Pipeline composition or triggering separate runs | approximate | SubDAGs are deprecated in Airflow. In ZenML, compose via Python functions or trigger separate pipeline runs. |

## Control Flow and Scheduling

| Airflow Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| Trigger rules (`all_success`, `all_done`, `one_failed`, etc.) | Limited / orchestrator-dependent | absent | Airflow trigger rules are a core scheduling primitive controlling when a task runs based on upstream states. ZenML has no equivalent trigger-rule lattice. **Always flag non-default trigger rules.** |
| Dynamic task mapping (`expand()`, Airflow 2.3+) | Fan-out patterns with limitations | approximate | ZenML cannot dynamically create steps from upstream outputs. Options: (a) static fan-out if cardinality known at pipeline construction, (b) dynamic pipelines (`@pipeline(dynamic=True)`) with `.map()`, or (c) multi-run pattern for runtime-determined cardinality. |
| Sensors (`BaseSensorOperator`, `poke`, `reschedule`, deferrable) | Polling step / schedule redesign / external triggers | absent | Airflow sensors have dedicated worker-slot and deferral semantics. ZenML has no sensor primitive. Modelling waits as steps changes resource consumption. |
| Scheduling (`schedule_interval`, cron presets) | `Schedule` object + orchestrator-backed schedules | approximate | Airflow's scheduler is native. ZenML attaches `Schedule` to a pipeline; actual scheduling depends on the orchestrator. |
| Timetables (Airflow 2.2+) | Cron/interval schedule + orchestrator semantics | approximate | Airflow timetables define data intervals and logical dates. ZenML scheduling is cron/interval — no "timetable API" equivalent. |
| Catchup / backfill | Orchestrator-dependent | approximate | Airflow creates runs for missed intervals when `catchup=True`. ZenML `Schedule(catchup=False)` is typical; backfill behavior depends on orchestrator. |
| Pools | Orchestrator/infra concurrency controls | absent | Airflow pools constrain concurrency across tasks. ZenML relies on orchestrator infrastructure for concurrency. **Flag if pools are used for correctness (rate limiting).** |
| Priority weights | Orchestrator/infra queueing | absent | No cross-step priority-weight API in ZenML. |
| SLAs (`sla`, `sla_miss_callback`) | External monitoring + timeouts | absent | No first-class SLA feature. Use timeouts and external monitoring/alerting. |

### Orchestrator scheduling support

Not all ZenML orchestrators support scheduling. Check this before migrating scheduled DAGs:

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

| Airflow Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| XCom (general) | ZenML artifacts (automatic data passing) | approximate | Airflow XCom is cross-task messaging (often DB-backed). ZenML artifacts are first-class persisted objects in the artifact store. Different serialization, size, mutability, and persistence semantics. |
| Return value → XCom push | Step return value → artifact | direct | In ZenML, step outputs are typed and materialized. Use `Annotated` for named outputs. |
| `ti.xcom_pull(task_ids=...)` | Direct function-call wiring in pipeline | direct | No pull needed — pass step outputs to step inputs in the `@pipeline` function. |
| XCom for control flow (branching, mapping cardinality) | Not directly supported | absent | Airflow can make scheduling decisions based on XCom values. ZenML cannot branch on artifact values at orchestration time. **Flag any control-plane XCom usage.** |
| `params` / `Param` | Pipeline parameters | approximate | ZenML distinguishes parameters (JSON-serializable literals) from artifacts (step outputs). Parameters are passed when calling the pipeline or via YAML config. |
| `dag_run.conf` | Run configuration / runtime pipeline args | approximate | Airflow supports runtime conf that can override params. ZenML equivalent is pipeline arguments at invocation time or trigger configurations. The override semantics differ. |
| Jinja templating in operator args | Not applicable | absent | Airflow templates like `{{ ds }}`, `{{ ti.xcom_pull(...) }}` have no ZenML equivalent. Replace with Python code: step context for run metadata, step inputs for data. |
| Variables | ZenML config (YAML) + secrets store | approximate | Airflow variables are global runtime key-value config. Migration should separate "secret" from "parameter" from "environment config." |
| Connections (`conn_id`) | Stack components + service connectors + secrets | approximate | Airflow connections are managed via UI/CLI/secrets backends. ZenML uses stack components for infrastructure and service connectors for credentials. Conceptual mapping is intent-based, not 1:1. |

## Error Handling and Monitoring

| Airflow Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `retries` | `StepRetryConfig(max_retries=N)` | direct | Step-level retry configuration. |
| `retry_delay` | `StepRetryConfig(delay=N)` | direct | Delay in seconds between retries. |
| `retry_exponential_backoff` | `StepRetryConfig(backoff=2)` | approximate | Airflow: boolean. ZenML: numeric factor. Pick an explicit factor (commonly 2). |
| `max_retry_delay` | Not directly supported | absent | ZenML has no cap on retry delay. If the Airflow DAG relies on this, note it in the migration report. |
| `execution_timeout` | Orchestrator/job-level timeouts | approximate | Airflow task timeouts are explicit per-task. ZenML timeouts are generally orchestrator settings (e.g., Kubernetes job timeout). Map carefully. |
| `on_success_callback` | `@step(on_success=my_hook)` | direct | ZenML hooks have different signatures than Airflow callbacks. No Airflow context dict. |
| `on_failure_callback` | `@step(on_failure=my_hook)` | direct | For notifications, use `alerter_failure_hook` (posts to stack's alerter). |
| Email alerts (`EmailOperator`) | ZenML alerters (Slack/Discord) or custom | approximate | ZenML's built-in alerters focus on chat services. Email requires custom integration. |

## Infrastructure and Operations

| Airflow Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `KubernetesPodOperator` | `@step(step_operator="kubernetes")` or Kubernetes orchestrator | approximate | Airflow runs arbitrary container images/commands. ZenML's Kubernetes step operator runs step code in pods via ZenML's containerization. "Pure external command" tasks are harder to map than "run Python code" tasks. |
| `DockerOperator` | ZenML containerization + image builder + container registry | approximate | ZenML builds Docker images for remote orchestrators/step operators. Use `DockerSettings` for configuration. |
| `SparkSubmitOperator` | `@step(step_operator="spark")` | approximate | ZenML has a Spark step operator (Spark on Kubernetes). Does not support dynamic pipelines with Spark. |
| DAG-level `default_args` | Pipeline-level settings / YAML config | approximate | Airflow `default_args` propagate to all tasks. ZenML uses pipeline-level `settings` and YAML config with step-level overrides. |
| Airflow metadata DB | ZenML server + artifact store | approximate | Airflow persists task state in its metadata DB. ZenML persists run/step/artifact metadata in the ZenML server and artifacts in the artifact store. |
| Airflow UI | ZenML dashboard | approximate | Different visualization and interaction paradigms. ZenML dashboard shows pipelines, runs, artifacts, models. |
| `airflow test` / `dag.test()` | Call pipeline function directly | approximate | ZenML local runs are "call the pipeline function." Also supports running under the active stack. |
