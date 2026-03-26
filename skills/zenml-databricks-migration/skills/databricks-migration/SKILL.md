---
name: databricks-to-zenml-migration
description: >-
  Migrate Databricks Workflows (Lakeflow Jobs) to idiomatic ZenML pipelines.
  Handles concept mapping (Job->pipeline, Task->step, task values->artifact),
  notebook refactoring, code translation for all Databricks task types
  (notebook_task, python_wheel_task, sql_task, dbt_task, condition_task,
  for_each_task, run_job_task, spark_jar_task), scheduling, retry config,
  compute mapping, and flags unsupported patterns (file arrival triggers,
  run_if semantics, shared cluster state, DBFS paths) for human review. Use
  this skill whenever the user mentions Databricks migration, converting
  Databricks Jobs or Workflows, porting workflows from Databricks, replacing
  Databricks orchestration with ZenML, or asks how a Databricks concept maps
  to ZenML -- even if they don't explicitly say "migrate". Also use when they
  paste Databricks job JSON or notebook code and ask to make it work with
  ZenML, or when they describe a workflow using Databricks terminology (task,
  job, notebook_task, dbutils, task values, job clusters, condition_task,
  for_each_task) in a ZenML context. If the user just asks a quick conceptual
  question ("what's the ZenML equivalent of dbutils.jobs.taskValues?"), answer
  it directly from the concept map -- no need to run the full migration workflow.
---

# Migrate Databricks Workflows to ZenML

This skill translates Databricks Workflows (Lakeflow Jobs) into idiomatic ZenML pipelines. It handles the full migration workflow: analyzing job definitions and notebook code, classifying each pattern, translating what maps cleanly, flagging what needs redesign, and producing a working ZenML project.

## How migration works at a high level

Databricks Workflows and ZenML look similar on the surface -- both define a DAG of tasks/steps with dependencies. But the underlying execution models are fundamentally different. Databricks models orchestration as an explicit DAG of **task objects** in JSON (each with a `task_key`, a concrete task type like `notebook_task` or `sql_task`, and an explicit `depends_on` list), with substantial **runtime configuration** co-located in task settings (compute binding, retries, notifications). ZenML models orchestration as **Python function calls** forming typed artifact edges, with runtime behavior driven by step/pipeline decorators, stack components, and containerization settings.

This means migration involves two distinct challenges:
1. **Structural translation**: mapping Databricks' JSON-defined DAG + heterogeneous task types into Python-defined ZenML steps and pipelines
2. **Semantic translation**: handling the differences in data passing (string-substituted task values vs typed artifacts), execution environment (managed Spark clusters vs containerized steps), and platform-coupled features (Unity Catalog, DBFS, dbutils)

### The three mapping types

Every Databricks concept falls into one of these categories:

| Type | Meaning | Action |
|------|---------|--------|
| **Direct** | Clean 1:1 mapping exists | Translate automatically |
| **Approximate** | Conceptual equivalent exists but semantics differ | Translate with caveats noted in migration report |
| **Absent** | No ZenML equivalent | Flag for human review with redesign suggestions |

See [references/concept-map.md](references/concept-map.md) for the full mapping tables.

## The Migration Workflow

### Phase 1: Receive and Analyze the Databricks Workflow

Ask the user for their Databricks job definition (JSON) and any associated notebook/script code. Databricks workflows come in multiple forms -- the user might provide:
- A **Jobs API 2.1 JSON** definition (the most complete representation)
- A **Databricks Asset Bundle YAML** (`databricks.yml` or resource YAML files)
- **Notebook code** (Python notebooks with `dbutils` calls, magics, widgets)
- A **mix of job JSON + notebook source files**

Read everything thoroughly before doing anything else. For each job, identify:

1. **Tasks and their types** -- What task types are used? (`notebook_task`, `python_wheel_task`, `spark_python_task`, `sql_task`, `dbt_task`, `spark_jar_task`, `condition_task`, `for_each_task`, `run_job_task`, `pipeline_task`)
2. **Dependencies** -- How are tasks wired? (`depends_on` with optional `outcome` conditions)
3. **Data flow** -- Where are `dbutils.jobs.taskValues` used? Are dynamic references (`{{tasks.<task>.values.<key>}}`) used in parameters?
4. **Control flow** -- Any `condition_task` nodes? `for_each_task` iteration? `run_if` settings beyond `ALL_SUCCESS`?
5. **Notebook analysis** -- For each `notebook_task`: does it use `dbutils.widgets`, `%sql`, `%pip`, `%run`, `display()`, Spark temp views, DBFS paths? (See the notebook classification guide in [references/gaps-and-flags.md](references/gaps-and-flags.md))
6. **Scheduling/triggers** -- Cron schedules? Periodic triggers? File arrival triggers? Continuous mode?
7. **Error handling** -- `max_retries`, `min_retry_interval_millis`, `timeout_seconds`, `retry_on_timeout`?
8. **Compute** -- Job clusters, existing clusters, serverless? Per-task library installs?
9. **Parameters** -- Job-level parameters with `{{job.parameters.*}}` references? Widget-based parameter passing?
10. **Access control** -- Job ACLs, Run-as identity, secret scopes?

### Phase 2: Classify and Plan

For each component identified in Phase 1, classify it using the mapping type (direct / approximate / absent). Use the decision logic below and the full tables in [references/concept-map.md](references/concept-map.md).

#### Quick classification guide

**Direct translations (translate automatically):**
- Multi-task job DAG structure → `@pipeline` with step calls matching `depends_on` edges
- `task_key` → step function name
- `max_retries` + `min_retry_interval_millis` → `StepRetryConfig(max_retries=N, delay=M)`
- `dbutils.jobs.taskValues.set()/get()` for simple data → step output/input artifacts
- `on_success_callback` / `on_failure` notifications → step hooks (`on_success`, `on_failure`)

**Approximate translations (translate with caveats):**
- `notebook_task` → `@step` wrapping refactored notebook logic (magics, dbutils, Spark session must be refactored)
- `python_wheel_task` → `@step` calling the wheel's entry point function directly (or containerized with Docker settings)
- `sql_task` → `@step` executing SQL via explicit client/connector (Databricks SQL connector + ZenML secrets)
- `dbt_task` → `@step` running dbt CLI in a container with explicit credentials
- `condition_task` → conditional pipeline logic (parameter-based for static pipelines, `@pipeline(dynamic=True)` + `.load()` for runtime values)
- `for_each_task` → `@pipeline(dynamic=True)` + `.map()` (concurrency is orchestrator-dependent)
- `run_job_task` → pipeline composition or API-triggered pipeline run
- Job parameters (`{{job.parameters.*}}`) → typed Python pipeline parameters
- Widget parameters (`dbutils.widgets.get()`) → step function parameters
- Cron scheduling → `Schedule(cron_expression=...)` (orchestrator-dependent)
- Job clusters / compute → `ResourceSettings` + orchestrator/step operator configuration
- `dbutils.secrets.get()` → ZenML secrets store
- Per-task libraries → `DockerSettings(requirements=[...])`

**Absent / needs redesign (flag for human review):**
- `run_if` with `ALL_DONE`, `AT_LEAST_ONE_FAILED`, etc. (ZenML has pipeline-level execution modes but not per-step `run_if`)
- File arrival triggers (Unity Catalog integration, no ZenML OSS equivalent)
- Continuous jobs (always-on streaming, not a ZenML pipeline pattern)
- Notebooks relying on `%run`, `%pip`, `%sql` magics, DBFS mounts, or shared Spark temp views across tasks
- Shared cluster state (cached tables, driver-local files, warm Spark context reused across tasks)
- DBFS-specific filesystem paths passed between tasks
- SQL/dbt tasks relying on Databricks-managed identity injection without portable auth

#### Present the migration plan

Before writing any code, present a summary to the user:

> "Here's what I found in your Databricks Workflow:
> - **Direct translations** (will migrate cleanly): [list]
> - **Approximate translations** (will work but with noted caveats): [list]
> - **Needs redesign** (cannot auto-migrate): [list with brief explanation]
>
> Shall I proceed with the migration?"

If there are HIGH-severity flags, explain each one concretely: what the Databricks code does, why ZenML can't replicate it directly, and what the recommended redesign looks like.

### Phase 3: Generate ZenML Code

Translate the Databricks Workflow into a ZenML project. Follow these conventions strictly.

#### Project structure

Every migrated project MUST use this layout:

```
migrated_pipeline/
├── steps/                    # One file per step
│   ├── extract.py
│   ├── transform.py
│   └── load.py
├── pipelines/
│   └── my_pipeline.py        # Pipeline definition
├── materializers/            # Custom materializers (if needed)
├── configs/
│   ├── dev.yaml
│   └── prod.yaml
├── run.py                    # CLI entry point (argparse, not click)
├── README.md
└── pyproject.toml
```

This matches the `zenml-pipeline-authoring` skill's conventions. Key rules:
- One step per file in `steps/`
- Separate pipeline definition from execution
- `run.py` uses `argparse` (click conflicts with ZenML)
- `pyproject.toml` with `zenml>=0.94.1` and `requires-python = ">=3.12"`
- Always generate `configs/dev.yaml` AND `configs/prod.yaml` (minimum two configs)
- Always generate a `README.md` explaining the migrated pipeline, how to run it, and what requires manual attention
- Include a brief ASCII DAG diagram in the pipeline file's module docstring showing the step dependency graph
- Run `zenml init` at project root

#### Translation patterns

For each Databricks task, apply the appropriate translation. See [references/code-patterns.md](references/code-patterns.md) for detailed side-by-side examples covering all major patterns.

**The core translation rule**: Extract the task's logic (from notebook cells, wheel entry points, or SQL files) into a `@step` function. Type-hint all inputs and outputs. Wire steps by passing outputs to inputs in the pipeline function.

```python
# Databricks: notebook_task with widget parameters
# dbutils.widgets.get("input_table") inside notebook

# ZenML: explicit typed parameters
@step
def extract(input_table: str, run_date: str) -> pd.DataFrame:
    # Replace Spark table read with portable data access
    return load_from_warehouse(input_table, run_date)
```

**Task values → Artifact passing**: Replace all `dbutils.jobs.taskValues.set()/get()` and `{{tasks.<task>.values.<key>}}` references with direct function-call wiring:

```python
# Databricks: task value set in producer, string-substituted in consumer
# dbutils.jobs.taskValues.set(key="count", value=42)
# Consumer: base_parameters: {"count": "{{tasks.producer.values.count}}"}

# ZenML: data flows naturally through function calls
@pipeline
def my_pipeline() -> None:
    count = producer_step()         # Returns int artifact
    consumer_step(count=count)      # Artifact passed directly
```

**Retries**: Map `max_retries` + `min_retry_interval_millis` to `StepRetryConfig`:

```python
# Databricks: "max_retries": 3, "min_retry_interval_millis": 60000
# ZenML:
@step(retry=StepRetryConfig(max_retries=3, delay=60, backoff=1))
def my_step() -> None: ...
```

**Notifications → Hooks**: Map task-level `email_notifications` / `webhook_notifications` to ZenML hooks:

```python
from zenml.hooks import alerter_failure_hook, alerter_success_hook

@step(on_failure=alerter_failure_hook, on_success=alerter_success_hook)
def my_step() -> None: ...
```

**Scheduling**: Map cron schedules to `Schedule`:

```python
from zenml.config.schedule import Schedule

# Databricks: quartz_cron_expression "0 0 2 * * ?" (Quartz 6-field)
# ZenML: standard 5-field cron
schedule = Schedule(cron_expression="0 2 * * *")
my_pipeline.with_options(schedule=schedule)()
```

Not all orchestrators support scheduling. Check [references/concept-map.md](references/concept-map.md) for the orchestrator support table.

#### Handling notebook tasks

Notebook tasks are the most common source of migration complexity. Follow this decision process:

1. **Scan for platform-coupled patterns**: `%run`, `%pip`, `%sql`, `%sh`, `dbutils.fs`, `display()`, Spark temp views shared across notebooks
2. **If the notebook is mostly pure Python** (uses `dbutils.widgets` for params and `dbutils.jobs.taskValues` for output, but otherwise standard Python/pandas): extract the logic into a `@step` function with typed parameters and return values
3. **If the notebook uses Spark heavily**: decide whether to keep Spark (use a step operator or Databricks orchestrator) or refactor to pandas/other (depends on data scale)
4. **If the notebook uses magics or shared state**: flag as HIGH-severity for manual refactoring. Add a `# TODO(migration)` comment explaining what needs manual attention

See the notebook classification guide in [references/gaps-and-flags.md](references/gaps-and-flags.md).

#### Code comment style

Keep migration-related comments concise and actionable. Use `# Migration note:` for brief inline caveats (1-2 lines) and `# TODO(migration):` for items requiring user action. Avoid lengthy paragraphs in code comments -- put detailed explanations in the migration report instead. The code should read as close to production-ready as possible; once the user has addressed the TODOs, the remaining comments should be minimal.

#### Handling approximate translations

When translating approximate patterns, add a brief inline comment noting the semantic difference:

```python
@step
def query_warehouse(query: str) -> pd.DataFrame:
    # Migration note: Databricks sql_task used managed identity injection and
    # warehouse_id binding. This step uses explicit credentials from ZenML
    # secrets. Ensure the SQL connector is configured and credentials are
    # stored via `zenml secret create`.
    import databricks.sql as dbsql
    # ... connect and query ...
```

#### Handling absent patterns

For patterns that have no ZenML equivalent, do NOT silently approximate them. Instead:

1. Add a clearly marked `# TODO(migration)` comment in the generated code
2. Include the pattern in the migration report
3. Suggest a redesign approach

```python
# TODO(migration): UNSUPPORTED -- Databricks run_if='ALL_DONE' on this task.
# ZenML does not support per-step run_if conditions. This task previously ran
# regardless of upstream success/failure. Consider: (a) using pipeline
# execution_mode=CONTINUE_ON_FAILURE + wrapping upstream steps in try/except,
# or (b) splitting into separate pipelines with independent failure domains.
@step
def cleanup_step(upstream_status: str) -> None:
    ...
```

### Phase 4: Produce the Migration Report

After generating the ZenML project, produce a `MIGRATION_REPORT.md` in the project root:

```markdown
# Migration Report: [Job Name] -> [Pipeline Name]

## Summary
- **Source**: Databricks Workflow `[job_name]`
- **Target**: ZenML pipeline `[pipeline_name]`
- **Tasks migrated**: X direct, Y approximate, Z flagged
- **Notebooks refactored**: N (of which M required manual attention)

## Direct Translations
| Databricks Task | ZenML Step | Notes |
|---|---|---|
| extract (notebook_task) | steps/extract.py | Widget params -> step args |

## Approximate Translations
| Databricks Task | ZenML Step | What Changed |
|---|---|---|
| train_wheel (python_wheel_task) | steps/train.py | Wheel entry point imported directly; DockerSettings for deps |
| query (sql_task) | steps/query.py | Uses explicit SQL connector + ZenML secrets instead of managed identity |

## Flagged for Review
| Databricks Pattern | Severity | Issue | Suggested Redesign |
|---|---|---|---|
| run_if='ALL_DONE' on cleanup | HIGH | No per-step run_if in ZenML | Use execution_mode=CONTINUE_ON_FAILURE + status artifacts |
| file_arrival trigger | HIGH | No ZenML OSS equivalent | Cloud events -> webhook -> pipeline trigger |
| %run notebook import | HIGH | Implicit code loading | Refactor into importable Python modules |

## Notebook Refactoring Summary
| Notebook | Detected Patterns | Refactor Status |
|---|---|---|
| /Repos/acme/etl/extract | widgets, taskValues | Auto-refactored |
| /Repos/acme/ml/train | %pip, Spark temp views, display() | Manual refactor required |

## Scheduling
- **Original**: Quartz cron `0 0 2 * * ?`, timezone `US/Pacific`
- **Migrated**: `Schedule(cron_expression='0 2 * * *')` -- requires orchestrator with scheduling support
- **Note**: Quartz 6-field cron converted to standard 5-field (seconds field dropped)

## Compute Mapping
| Databricks Cluster | ZenML Equivalent | Notes |
|---|---|---|
| cpu_cluster (i3.xlarge, 2 workers) | ResourceSettings(cpu_count=2, memory="8Gi") | Spark cluster lifecycle differs |
| gpu_cluster (g5.2xlarge) | ResourceSettings(gpu_count=1, memory="16Gi") | GPU scheduling is orchestrator-dependent |

## Limitations and Key Differences
[Summarize the most important behavioral differences the user should be aware of. Put this BEFORE the "What You Get for Free" section so the user sees caveats before benefits.]

## What's NOT Migrated
[List Databricks platform features outside the job definition: secret scopes, Unity Catalog governance, DBFS mounts, workspace permissions, etc., with guidance on the ZenML equivalent]

## What You Get for Free After Migration
ZenML provides capabilities that Databricks Workflows do not have natively:
- **Artifact versioning and lineage** -- every step output is versioned and traceable
- **Step caching** -- skip re-execution when code and inputs haven't changed
- **Stack abstraction** -- same pipeline code runs on local, K8s, Vertex, SageMaker by switching stacks
- **Model Control Plane** -- track ML models with versioning and promotion stages
- **Service connectors** -- unified cloud auth with automatic token refresh
- **Pipeline execution modes** -- control failure behavior (FAIL_FAST, CONTINUE_ON_FAILURE)
- **Typed artifacts** -- full datasets/models, not just 48KiB JSON blobs

## Recommended Next Steps
1. Run the `zenml-quick-wins` skill for metadata logging, experiment tracking, and alerters
2. Install the ZenML docs MCP server: `claude mcp add zenmldocs --transport http https://docs.zenml.io/~gitbook/mcp`
3. [Specific links to docs for each flagged pattern]
4. For Docker settings, YAML config, or deployment: use the `zenml-pipeline-authoring` skill
```

### Phase 5: Suggest Next Steps

After migration is complete, always include a "Recommended Next Steps" section in the migration report AND communicate it to the user.

#### 1. Run the `zenml-quick-wins` skill

Always suggest this as the immediate next step:

> "Now that the migration is done, I'd recommend running the `zenml-quick-wins` skill to add metadata logging, experiment tracking, and other production features to your pipeline."

#### 2. Documentation links for flagged patterns

For every flagged pattern, include a link to the relevant ZenML documentation:

- Scheduling: `https://docs.zenml.io/how-to/steps-pipelines/schedule-a-pipeline`
- Service connectors: `https://docs.zenml.io/how-to/infrastructure-deployment/auth-management`
- Dynamic pipelines: `https://docs.zenml.io/how-to/steps-pipelines/dynamic-pipelines`
- Orchestrators: `https://docs.zenml.io/stacks/stack-components/orchestrators`
- Triggers: `https://docs.zenml.io/how-to/steps-pipelines/trigger-a-pipeline`
- Containerization/Docker: `https://docs.zenml.io/how-to/containerization/containerization`
- Secrets management: `https://docs.zenml.io/how-to/project-setup-and-management/secret-management`

#### 3. Suggest installing the ZenML docs MCP server

> "For easier access to ZenML documentation while you work, you can install the ZenML docs MCP server: `claude mcp add zenmldocs --transport http https://docs.zenml.io/~gitbook/mcp`"

#### 4. Community support for unsupported patterns

When the migration has HIGH-severity flags -- patterns that couldn't be directly migrated -- offer to help the user get support from the ZenML community. **When there are 2+ HIGH-severity flags**, generate a pre-made Slack message for `zenml.io/slack`. Include enough context that a ZenML engineer can understand the full situation: what's being migrated, the specific unsupported patterns with code snippets, what workarounds were suggested, and what the user is looking for:

```markdown
**Databricks -> ZenML Migration Help**

I'm migrating a Databricks Workflow (`[job_name]`) that uses [patterns]. The migration skill flagged these as needing redesign:

1. **[Pattern]**: [brief description + code snippet showing the Databricks config]
   - Suggested workaround: [X]
   - Why this matters: [what behavior would change without a proper solution]

2. **[Pattern]**: [brief description + code snippet]
   - Suggested workaround: [Y]

I've implemented the workarounds above, but I'm wondering if there's a better approach, an upcoming feature, or a pattern I'm missing. Happy to share the full migration report if helpful.
```

#### 5. Open GitHub issues for genuine feature gaps

When the migration reveals a **genuine missing feature** in ZenML (not just a "this works differently" situation, but a real capability gap that multiple users would benefit from), offer to open a GitHub issue on `zenml-io/zenml` using `gh issue create`. Include the Databricks pattern, the attempted workaround, and why the feature would be valuable. This helps the ZenML team prioritize features that real migration users need.

#### 6. Run `/simplify` to clean up the migrated code

After migration is complete, always suggest running the `/simplify` skill on the generated code. Migration often produces verbose comments, redundant patterns, and opportunities for consolidation. `/simplify` will review the code for reuse opportunities, quality issues, and efficiency improvements -- helping the migrated code feel more like production code and less like a translation artifact.

> "The migration is done. I'd recommend running `/simplify` on the generated code to clean up migration comments, reduce duplication, and ensure the code follows ZenML best practices."

#### 7. Further customization via `zenml-pipeline-authoring`

The `zenml-pipeline-authoring` skill handles deeper customization:
- **Docker settings** for remote execution (Kubernetes/Vertex/SageMaker)
- **YAML configuration** for multi-environment setups
- **Custom materializers** for domain-specific types
- **Pipeline deployment** for HTTP serving

## Important Behavioral Differences to Communicate

These are the most common sources of confusion after migration. Always mention the relevant ones in the migration report.

### Task values != Artifacts

Databricks task values (`dbutils.jobs.taskValues`) are small JSON blobs (48KiB limit) used as a control channel between tasks. ZenML artifacts are first-class persisted objects stored in the artifact store. This changes:
- **Serialization**: ZenML uses materializers (type-specific serializers), not JSON strings
- **Size**: Artifacts can be arbitrarily large (DataFrames, models, images)
- **Lifecycle**: Artifacts are versioned and persist across runs; task values are ephemeral
- **Caching**: ZenML can skip re-execution when inputs haven't changed

### Execution model

Databricks tasks run on managed Spark clusters (shared or per-task). ZenML steps run in containers managed by the orchestrator. This means:
- No shared Spark context or cluster-level caches between steps
- No warm driver memory or `/tmp` handoffs between steps
- No notebook kernel environment (no `dbutils`, no magics, no `display()`)
- Step isolation is stronger -- each step is its own container

### Parameterization: string substitution vs typed values

Databricks uses `{{...}}` string substitution for dynamic references -- syntax errors can be silently ignored. ZenML pipeline/step parameters are real typed Python values. Migration must explicitly parse and type-cast all dynamic references.

### Scheduling and triggers

Databricks has first-class triggers for cron, file arrival, table update, and continuous jobs. ZenML delegates scheduling to the orchestrator -- not all orchestrators support it. File arrival triggers and continuous jobs have no ZenML OSS equivalent and require external eventing infrastructure.

## Anti-Patterns in Migration

| Anti-pattern | Why it's wrong | What to do instead |
|---|---|---|
| Keeping `dbutils.jobs.taskValues` calls | ZenML has no dbutils context | Wire data through step inputs/outputs |
| Keeping `dbutils.widgets.get()` calls | No notebook kernel in ZenML steps | Use step function parameters |
| Keeping `%sql` / `%pip` / `%run` magics | Not valid Python in containerized steps | Refactor to explicit Python (SQL client, Docker deps, module imports) |
| Passing DBFS paths between steps | DBFS paths don't exist outside Databricks | Pass data as artifacts or use cloud storage URIs |
| Translating `condition_task` to static if/else when condition depends on runtime values | Static pipelines can't branch on step outputs | Use `@pipeline(dynamic=True)` with `.load()` |
| Ignoring `run_if` during migration | Silently changes failure handling behavior | Always flag non-default `run_if` settings |
| Translating `for_each_task` to a Python for loop | Loses per-item parallelism and observability | Use dynamic pipelines with `.map()` |
| Assuming shared cluster state | ZenML steps are isolated containers | Pass all data through artifacts, not shared memory |
| Keeping Databricks-specific auth (managed tokens) | Won't work outside Databricks | Use ZenML secrets + service connectors |

## References

### Detailed reference files

- [references/concept-map.md](references/concept-map.md) -- Full concept mapping tables (40+ Databricks concepts -> ZenML equivalents), task type mappings, and stack component mappings
- [references/code-patterns.md](references/code-patterns.md) -- Side-by-side code translations for all major patterns (linear notebook workflow, branching, task values, for_each, mixed task types, retries/timeouts, job clusters, run_job_task, file arrival triggers)
- [references/gaps-and-flags.md](references/gaps-and-flags.md) -- Behavioral differences, unsupported patterns, notebook classification guide, migration decision tree, and the full list of "refuse to auto-migrate" patterns

### ZenML documentation

For topics beyond migration (stack setup, experiment tracking, deployment), query the ZenML docs at https://docs.zenml.io.
