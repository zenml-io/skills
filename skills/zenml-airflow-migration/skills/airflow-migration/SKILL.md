---
name: airflow-to-zenml-migration
description: >-
  Migrate Apache Airflow DAGs, operators, and workflows to idiomatic ZenML
  pipelines. Handles concept mapping (DAG→pipeline, operator→step, XCom→artifact),
  code translation, scheduling, retry config, Docker settings, and flags
  unsupported patterns (trigger rules, sensors, dynamic task mapping) for human
  review. Use this skill whenever the user mentions Airflow migration, converting
  Airflow DAGs, porting workflows from Airflow, replacing Airflow with ZenML, or
  asks how an Airflow concept maps to ZenML — even if they don't explicitly say
  "migrate". Also use when they paste Airflow code and ask to make it work with
  ZenML, or when they describe a workflow using Airflow terminology (DAG, operator,
  XCom, sensor, task group) in a ZenML context. If the user just asks a quick
  conceptual question ("what's the ZenML equivalent of XCom?"), answer it directly
  from the concept map — no need to run the full migration workflow.
---

# Migrate Airflow to ZenML

This skill translates Apache Airflow DAGs into idiomatic ZenML pipelines. It handles the full migration workflow: analyzing Airflow code, classifying each pattern, translating what maps cleanly, flagging what needs redesign, and producing a working ZenML project.

## How migration works at a high level

Airflow and ZenML look similar on the surface — DAG maps to pipeline, operator maps to step, XCom maps to artifact — but their execution models are fundamentally different. Airflow is built around a scheduler-backed, database-persisted task-instance state machine. ZenML is built around artifact lineage, stack-driven infrastructure abstraction, and Python-first pipeline composition.

This means migration is not a rename-the-primitives exercise. Some patterns translate directly, some need approximation, and some require genuine redesign. The skill's job is to be honest about which is which.

### The three mapping types

Every Airflow concept falls into one of these categories:

| Type | Meaning | Action |
|------|---------|--------|
| **Direct** | Clean 1:1 mapping exists | Translate automatically |
| **Approximate** | Conceptual equivalent exists but semantics differ | Translate with caveats noted in migration report |
| **Absent** | No ZenML equivalent | Flag for human review with redesign suggestions |

See [references/concept-map.md](references/concept-map.md) for the full mapping tables.

## The Migration Workflow

### Phase 1: Receive and Analyze the Airflow Code

Ask the user for their Airflow DAG file(s). Read the code thoroughly before doing anything else. For each DAG, identify:

1. **Tasks and their types** — What operators are used? (`PythonOperator`, `BashOperator`, `KubernetesPodOperator`, sensors, custom operators, TaskFlow `@task`)
2. **Dependencies** — How are tasks wired? (`>>`, `set_upstream`, TaskFlow data passing)
3. **Data flow** — Where is XCom used? Is it for data passing or control flow decisions?
4. **Control flow** — Any branching (`BranchPythonOperator`), short-circuiting, trigger rules beyond `all_success`?
5. **Dynamic patterns** — Any `expand()` / dynamic task mapping?
6. **Scheduling** — `schedule_interval`, cron presets, timetables, catchup settings?
7. **Error handling** — Retries, timeouts, callbacks, SLAs?
8. **External dependencies** — Connections, variables, sensors waiting on external systems?
9. **Infrastructure** — Kubernetes pods, Spark jobs, custom Docker images?

### Phase 2: Classify and Plan

For each component identified in Phase 1, classify it using the mapping type (direct / approximate / absent). Use the decision logic below and the full tables in [references/concept-map.md](references/concept-map.md).

#### Quick classification guide

**Direct translations (translate automatically):**
- `PythonOperator` / `@task` → `@step`
- Task dependencies via data passing → step invocation order with artifact wiring
- Return-value XCom → step output artifacts
- `retries` / `retry_delay` → `StepRetryConfig`
- `on_success_callback` / `on_failure_callback` → step hooks (`on_success`, `on_failure`)
- TaskFlow API function composition → ZenML step composition (nearly identical syntax)

**Approximate translations (translate with caveats):**
- `BashOperator` → `@step` with `subprocess.run()` (containerization differs on remote stacks)
- `BranchPythonOperator` → conditional pipeline logic, **but only if branching depends on pipeline parameters, not upstream step outputs**
- XCom for data passing → artifact passing (different persistence, serialization, and lifecycle semantics)
- `params` / `dag_run.conf` → pipeline parameters / run configuration
- Scheduling → `Schedule` object (depends on orchestrator support)
- Connections → stack components + service connectors + secrets
- Variables → ZenML config + secrets store
- TaskGroups → Python composition functions (no UI grouping equivalent)
- `retry_exponential_backoff` → `StepRetryConfig(backoff=2)` (boolean → numeric factor)
- `KubernetesPodOperator` → `@step(step_operator="kubernetes")` (ZenML containerization, not arbitrary container commands)

**Absent / needs redesign (flag for human review):**
- Non-default trigger rules (`all_done`, `one_failed`, `none_skipped`, etc.)
- Branching based on upstream task outputs (not pipeline parameters)
- Dynamic task mapping where the iterable comes from an upstream task (`expand()` over runtime data)
- Sensors with `reschedule` mode or deferrable operators
- Pools and priority weights used for correctness (rate limiting)
- SLA monitoring (`sla`, `sla_miss_callback`)
- SubDagOperator-based control flow

#### Present the migration plan

Before writing any code, present a summary to the user:

> "Here's what I found in your Airflow DAG:
> - **Direct translations** (will migrate cleanly): [list]
> - **Approximate translations** (will work but with noted caveats): [list]
> - **Needs redesign** (cannot auto-migrate): [list with brief explanation]
>
> Shall I proceed with the migration?"

If there are HIGH-severity flags, explain each one concretely: what the Airflow code does, why ZenML can't replicate it directly, and what the recommended redesign looks like.

### Phase 3: Generate ZenML Code

Translate the Airflow DAG into a ZenML project. Follow these conventions strictly.

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
- Run `zenml init` at project root

#### Translation patterns

For each Airflow task, apply the appropriate translation. See [references/code-patterns.md](references/code-patterns.md) for detailed side-by-side examples covering all major patterns.

**The core translation rule**: Move the task's callable body into a `@step` function. Type-hint all inputs and outputs. Wire steps by passing outputs to inputs in the pipeline function.

```python
# Airflow
def extract() -> list[int]:
    return [1, 2, 3]

t = PythonOperator(task_id="extract", python_callable=extract)

# ZenML
@step
def extract() -> List[int]:
    return [1, 2, 3]
```

**XCom → Artifact passing**: Replace all `ti.xcom_pull()` / `ti.xcom_push()` with direct function-call wiring:

```python
# Airflow: explicit XCom pull via templating
sum_ = PythonOperator(
    task_id="sum",
    python_callable=sum_numbers,
    op_kwargs={"numbers": "{{ ti.xcom_pull(task_ids='extract') }}"},
)

# ZenML: data flows naturally through function calls
@pipeline
def my_pipeline() -> None:
    numbers = extract()
    total = sum_numbers(numbers)  # Artifact passed directly
```

**Retries**: Map `retries` + `retry_delay` + `retry_exponential_backoff` to `StepRetryConfig`:

```python
# Airflow
default_args = {"retries": 3, "retry_delay": timedelta(seconds=10), "retry_exponential_backoff": True}

# ZenML
@step(retry=StepRetryConfig(max_retries=3, delay=10, backoff=2))
def my_step() -> None: ...
```

**Callbacks → Hooks**: Map `on_failure_callback` / `on_success_callback` to ZenML hooks. For chat notifications, use ZenML's standard alerter hooks:

```python
from zenml.hooks import alerter_failure_hook, alerter_success_hook

@step(on_failure=alerter_failure_hook, on_success=alerter_success_hook)
def my_step() -> None: ...
```

**Scheduling**: Map `schedule_interval` or cron presets to `Schedule`:

```python
from zenml.config.schedule import Schedule

schedule = Schedule(cron_expression="0 2 * * *")  # Was schedule="@daily" or "0 2 * * *"
my_pipeline.with_options(schedule=schedule)()
```

Not all orchestrators support scheduling. Check [references/concept-map.md](references/concept-map.md) for the orchestrator support table.

#### Handling approximate translations

When translating approximate patterns, always add a comment in the generated code explaining the semantic difference. This helps the user understand what changed and why.

```python
@step
def run_shell_command(cmd: str) -> str:
    # Migration note: Airflow's BashOperator ran in the Airflow worker environment.
    # This step runs inside a container on the active orchestrator. Working directory
    # and available system tools may differ. Verify the command works in your target
    # stack's container environment.
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True, check=True)
    return result.stdout
```

#### Handling absent patterns

For patterns that have no ZenML equivalent, do NOT silently approximate them. Instead:

1. Add a clearly marked `# TODO(migration)` comment in the generated code
2. Include the pattern in the migration report
3. Suggest a redesign approach

```python
# TODO(migration): UNSUPPORTED — Airflow trigger rule 'all_done' on this step.
# ZenML does not support trigger rules. This step previously ran regardless of
# upstream success/failure. Consider: (a) splitting into separate pipelines with
# independent failure domains, or (b) wrapping upstream steps in try/except and
# using a status artifact to communicate outcome.
@step
def join_step(upstream_status: str) -> None:
    ...
```

### Phase 4: Produce the Migration Report

After generating the ZenML project, produce a `MIGRATION_REPORT.md` in the project root. This is the user's map of everything that changed, approximated, or needs attention.

```markdown
# Migration Report: [DAG Name] → [Pipeline Name]

## Summary
- **Source**: Airflow DAG `[dag_id]`
- **Target**: ZenML pipeline `[pipeline_name]`
- **Tasks migrated**: X direct, Y approximate, Z flagged

## Direct Translations
| Airflow Task | ZenML Step | Notes |
|---|---|---|
| extract (PythonOperator) | steps/extract.py | Clean translation |

## Approximate Translations
| Airflow Task | ZenML Step | What Changed |
|---|---|---|
| run_cmd (BashOperator) | steps/run_cmd.py | Now runs in container; verify command works in target environment |

## Flagged for Review
| Airflow Pattern | Severity | Issue | Suggested Redesign |
|---|---|---|---|
| trigger_rule='all_done' on join_step | HIGH | No ZenML equivalent | Split into independent pipelines or use status artifacts |

## Scheduling
- **Original**: `schedule='@daily'`, catchup=False
- **Migrated**: `Schedule(cron_expression='0 0 * * *')` — requires orchestrator with scheduling support

## What's NOT Migrated
[List any Airflow infrastructure that lives outside the DAG: connections, variables, pools, etc., with guidance on the ZenML equivalent]

## What You Get for Free After Migration
ZenML provides capabilities that Airflow does not have natively:
- **Artifact versioning and lineage** — every step output is versioned and traceable
- **Step caching** — skip re-execution when code and inputs haven't changed
- **Stack abstraction** — same pipeline code runs on local, K8s, Vertex, SageMaker by switching stacks
- **Model Control Plane** — track ML models with versioning and promotion stages
- **Service connectors** — unified cloud auth with automatic token refresh

## Recommended Next Steps
1. Run the `zenml-quick-wins` skill for metadata logging, experiment tracking, and alerters
2. Install the ZenML docs MCP server: `claude mcp add zenmldocs --transport http https://docs.zenml.io/~gitbook/mcp`
3. [Specific links to docs for each flagged pattern]
4. For Docker settings, YAML config, or deployment: use the `zenml-pipeline-authoring` skill
```

### Phase 5: Suggest Next Steps

After migration is complete, always include a "Recommended Next Steps" section in the migration report AND communicate it to the user. This section should cover three things:

#### 1. Run the `zenml-quick-wins` skill

Always suggest this as the immediate next step. The quick-wins skill adds production-readiness features that complement the migration: metadata logging, experiment tracking, alerter setup, secrets management, and Model Control Plane configuration. Tell the user:

> "Now that the migration is done, I'd recommend running the `zenml-quick-wins` skill to add metadata logging, experiment tracking, and other production features to your pipeline."

#### 2. Documentation links for flagged patterns

For every flagged pattern in the migration report, include a link to the relevant ZenML documentation page. Don't just say "set up a trigger" — link to the specific docs page. Common links to include:

- Scheduling: `https://docs.zenml.io/how-to/steps-pipelines/schedule-a-pipeline`
- Service connectors (for auth): `https://docs.zenml.io/how-to/infrastructure-deployment/auth-management`
- Dynamic pipelines: `https://docs.zenml.io/how-to/steps-pipelines/dynamic-pipelines`
- Orchestrators (general): `https://docs.zenml.io/stacks/stack-components/orchestrators`
- Triggers and event-driven pipelines: `https://docs.zenml.io/how-to/steps-pipelines/trigger-a-pipeline`

#### 3. Suggest installing the ZenML docs MCP server

ZenML has a documentation MCP server that provides real-time lookups from the docs. This is especially valuable post-migration when the user needs to look up ZenML-specific patterns. Suggest:

> "For easier access to ZenML documentation while you work, you can install the ZenML docs MCP server: `claude mcp add zenmldocs --transport http https://docs.zenml.io/~gitbook/mcp`"

#### 4. Community support for unsupported patterns

When the migration has HIGH-severity flags — patterns that couldn't be directly migrated — offer to help the user get support from the ZenML community. This is important because the ZenML engineering team often has workarounds, opinions, or may even build features to support missing patterns.

**When there are 2+ HIGH-severity flags**, generate a pre-made Slack message that the user can post to the ZenML community Slack (`zenml.io/slack`). The message should include:
- A brief description of what they're migrating (e.g., "Migrating an Airflow DAG with dynamic task mapping and sensors")
- The specific unsupported patterns, with short code snippets showing the Airflow code
- What they've already tried (the redesign approaches from the migration report)
- A clear ask: "Any suggestions for better approaches?"

Format it as a fenced code block the user can copy-paste:

```markdown
**Airflow → ZenML Migration Help**

I'm migrating an Airflow DAG that uses [patterns]. The migration skill flagged these as needing redesign:

1. **[Pattern]**: [brief description + code snippet]
2. **[Pattern]**: [brief description + code snippet]

The suggested workarounds are [X], but I'm wondering if there's a better approach or upcoming feature that could help.
```

Optionally, offer to create an unlisted GitHub gist (`gh gist create --public=false`) containing the original Airflow code and the migration report, so the community has full context.

#### 5. Further customization via `zenml-pipeline-authoring`

The `zenml-pipeline-authoring` skill handles deeper customization:

- **Docker settings** for remote execution (Kubernetes/Vertex/SageMaker)
- **YAML configuration** for multi-environment setups
- **Custom materializers** for domain-specific types
- **Pipeline deployment** for HTTP serving

## Important Behavioral Differences to Communicate

These are the most common sources of confusion after migration. Always mention the relevant ones in the migration report.

### XCom ≠ Artifacts

Airflow XCom is lightweight message passing (often DB-backed, small values). ZenML artifacts are first-class persisted objects stored in the artifact store. This changes:
- **Serialization**: ZenML uses materializers (type-specific serializers), not JSON/pickle
- **Size**: Artifacts can be arbitrarily large (DataFrames, models, images)
- **Lifecycle**: Artifacts are versioned and persist across runs; XCom is ephemeral per run
- **Caching**: ZenML can skip re-execution when inputs haven't changed

### Execution model

Airflow tasks run in worker processes managed by a central scheduler. ZenML steps run in containers managed by the orchestrator (Kubernetes pods, Vertex AI jobs, etc.). This means:
- No shared filesystem between steps on remote orchestrators
- No Airflow context dict (`context["ti"]`, `context["dag_run"]`, etc.)
- Step isolation is stronger — each step is its own process/container

### Scheduling ownership

Airflow owns scheduling natively (the scheduler is a core component). ZenML delegates scheduling to the orchestrator — not all orchestrators support it, and the scheduling semantics depend on the orchestrator. Always check orchestrator compatibility.

## Anti-Patterns in Migration

| Anti-pattern | Why it's wrong | What to do instead |
|---|---|---|
| Keeping `ti.xcom_pull()` calls | ZenML has no task instance context | Wire data through step inputs/outputs |
| Passing file paths between steps | Works locally, breaks on remote orchestrators | Pass data as artifacts (DataFrames, dicts, etc.) |
| Translating `BranchPythonOperator` that branches on task outputs | ZenML can't branch on artifact values at graph construction | Redesign: run all branches but no-op when condition is false, or split into separate pipelines |
| Mapping `expand()` over upstream output to a simple loop | Loses Airflow's task-level retry/observability per item | Use ZenML dynamic pipelines (if cardinality known) or multi-run pattern (if runtime-determined) |
| Ignoring trigger rules during migration | Silently changes pipeline behavior | Always flag non-default trigger rules; never drop them without user awareness |
| Translating sensors to `time.sleep()` loops | Consumes compute slot for entire wait | Consider orchestrator scheduling, external triggers, or polling steps with timeouts |
| Replicating Airflow's `params` override behavior | `dag_run.conf` override semantics are Airflow-specific | Use ZenML pipeline parameters with explicit precedence (YAML config > defaults) |

## References

### Detailed reference files

- [references/concept-map.md](references/concept-map.md) — Full concept mapping tables (30+ Airflow concepts → ZenML equivalents) and stack component mappings
- [references/code-patterns.md](references/code-patterns.md) — Side-by-side code translations for all major patterns (linear DAG, branching, XCom, TaskFlow, dynamic mapping, retries, sensors, TaskGroups, runtime params, KubernetesPodOperator, callbacks)
- [references/gaps-and-flags.md](references/gaps-and-flags.md) — Behavioral differences, unsupported patterns, migration decision tree, and the full list of "refuse to auto-migrate" patterns

### ZenML documentation

For topics beyond migration (stack setup, experiment tracking, deployment), query the ZenML docs at https://docs.zenml.io.
