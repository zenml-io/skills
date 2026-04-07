---
name: prefect-to-zenml-migration
description: >-
  Migrate Prefect flows, tasks, and deployment patterns to idiomatic ZenML
  pipelines. Handles concept mapping (`@flow`→`@pipeline`, `@task`→`@step`,
  result persistence→artifacts), dynamic-execution analysis, code translation,
  scheduling, retries, Blocks/secrets decomposition, and flags unsupported
  patterns (`allow_failure()`, `return_state=True`, pause/suspend, global
  concurrency, task-runner semantics) for human review. Use this skill whenever
  the user mentions Prefect migration, converting Prefect flows, porting
  workflows from Prefect, replacing Prefect with ZenML, or asks how a Prefect
  concept maps to ZenML — even if they do not explicitly say "migrate". Also
  use when they paste Prefect code and ask to make it work with ZenML, or when
  they describe a workflow using Prefect terminology (`@flow`, `@task`,
  `.submit()`, `.map()`, `State`, Blocks, Deployments, work pools, Automations)
  in a ZenML context. If the user asks a quick conceptual question ("what is
  the ZenML equivalent of a Prefect Block?"), answer it directly from the
  concept map — no need to run the full migration workflow.
---

# Migrate Prefect to ZenML

This skill translates Prefect flows into idiomatic ZenML pipelines. It handles the full migration workflow: analyzing Prefect code, classifying each pattern, translating what maps cleanly, flagging what needs redesign, and producing a working ZenML project plus a migration report.

## How migration works at a high level

Prefect and ZenML look similar at first glance because both are Python-first and both decorate functions. But the execution story is different.

- **Prefect** runs the flow body like regular Python at runtime. That means the flow can branch on task outputs, inspect states, submit new tasks dynamically, and use pause/suspend semantics while the run is already in progress.
- **ZenML static pipelines** compile a DAG before steps run. Step outputs are versioned artifacts, not ordinary in-memory values available during pipeline construction.
- **ZenML dynamic pipelines** recover part of Prefect's runtime flexibility, but they are still an approximation rather than a full state-model match. As of the current ZenML docs, dynamic pipelines are explicitly marked **experimental** and support only a subset of orchestrators.

So migration is never just "rename `@flow` to `@pipeline`". The real job is to decide which Prefect behaviors:

1. map cleanly,
2. map with semantic differences, or
3. require redesign.

### The three mapping types

Every Prefect concept falls into one of these categories:

| Type | Meaning | Action |
|------|---------|--------|
| **Direct** | Clean or near-clean mapping exists | Translate automatically |
| **Approximate** | Similar concept exists, but behavior changes | Translate with caveats in the migration report |
| **Absent** | No trustworthy ZenML equivalent | Flag for human review with redesign suggestions |

See [references/concept-map.md](references/concept-map.md) for the full mapping tables.

## The Migration Workflow

### Phase 1: Receive and Analyze the Prefect Code

Ask the user for their Prefect flow files, deployment config, and any supporting modules. Read everything before writing code. For each workflow, identify:

1. **Flows and tasks** — Which functions use `@flow` and `@task`? Are there nested flows or direct task calls?
2. **Execution model** — Is the workflow shape known up front, or does it depend on runtime task outputs?
3. **Dynamic control flow** — Any `if` / `for` logic that branches on task results, or any `.submit()` / `.map()` fan-out?
4. **State handling** — Any `return_state=True`, manual `State` inspection, returning `Failed(...)`, or `allow_failure()`?
5. **Concurrency model** — Which task runner is used (`ThreadPoolTaskRunner`, `ProcessPoolTaskRunner`, Dask, Ray)? Is correctness tied to that runner?
6. **Caching and result persistence** — Any `cache_key_fn`, cache expiration, `persist_result`, custom serializers, or storage blocks?
7. **Human-in-the-loop** — Any `pause_flow_run()` or `suspend_flow_run()` behavior?
8. **Configuration** — Any Prefect Blocks, Variables, secrets, or `prefect.yaml` deployment config?
9. **Deployment and automation** — Any Deployments, work pools, workers, schedules, Automations, or webhook/event triggers?
10. **Transactions or rollback hooks** — Any `on_commit`, `on_rollback`, or other transactional semantics?

### Phase 2: Classify and Plan

For each component identified in Phase 1, classify it as direct / approximate / absent using the logic below and the full tables in [references/concept-map.md](references/concept-map.md).

#### Quick classification guide

**Direct or near-direct translations (translate automatically):**
- Simple `@task` → `@step`
- Simple static `@flow` → `@pipeline`
- Task return values used for data passing → artifact passing
- Simple task retries → `StepRetryConfig`
- Secret-only Blocks → ZenML secrets

**Approximate translations (translate with caveats):**
- `@flow` generally → `@pipeline` (execution model differs)
- Nested flows → pipeline composition
- `.submit()` / `.map()` → dynamic pipelines or orchestrator-driven parallelism
- Blocks → split into secrets + service connectors + stack settings + YAML config
- Deployments / work pools → schedules + orchestrator choice + runtime config
- Simple cron-like schedules → `Schedule(...)` when the target orchestrator supports scheduling
- Result persistence / serializers → artifacts + materializers
- Flow/task hooks → ZenML hooks and alerters
- Pause / suspend → dynamic waits, explicit approval steps, or split workflows

**Absent / redesign-required patterns (flag for human review):**
- `allow_failure()`
- `return_state=True`
- Manual `State` inspection or returning `Failed(...)`
- Global concurrency limits and `rate_limit()`
- Task-runner semantics relied on for correctness
- Transactions / rollback hooks
- Push work pools or tightly coupled Prefect Cloud control-plane logic
- Custom cache keys / TTL where the business semantics depend on them

#### Present the migration plan

Before generating code, present a concrete summary:

> "Here's what I found in your Prefect workflow:
> - **Direct translations** (will migrate cleanly): [list]
> - **Approximate translations** (will work with caveats): [list]
> - **Needs redesign** (cannot be trusted as an automatic migration): [list]
>
> Shall I proceed with the migration?"

If there are HIGH-severity flags, explain them in plain language:

- what the Prefect code currently does,
- why ZenML cannot reproduce it directly, and
- what the recommended redesign looks like.

### Phase 3: Generate ZenML Code

Translate the Prefect project into an idiomatic ZenML project. Follow these conventions strictly.

#### Project structure

Every migrated project MUST use this layout:

```text
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

Key rules:
- One step per file in `steps/`
- Separate pipeline definition from execution
- `run.py` uses `argparse`
- `pyproject.toml` with `zenml>=0.94.1` and `requires-python = ">=3.12"`
- Always generate `configs/dev.yaml` and `configs/prod.yaml`
- Always generate a `README.md` explaining what was migrated and what still needs manual attention
- Run `zenml init` at project root

#### Core translation rules

See [references/code-patterns.md](references/code-patterns.md) for side-by-side examples.

**1. Prefer static pipelines by default**

A ZenML static pipeline is the safest default when the DAG shape is known before execution.

**2. Use `@pipeline(dynamic=True)` only when the Prefect flow truly depends on runtime outputs**

Dynamic pipelines are the closest ZenML equivalent for:
- branching on step outputs,
- runtime fan-out over same-run artifacts,
- runtime-shaped workflows.

But they are not a universal substitute for Prefect's state model. When dynamic pipelines are needed, call that out clearly in the migration report.

**3. Treat failure/state features as a data-model redesign, not a scheduling trick**

For `allow_failure()` and `return_state=True`, do **not** silently replace them with a global execution mode. Instead, redesign around explicit outputs such as:

```python
{"ok": bool, "value": ..., "error": str | None}
```

That makes the new behavior visible and testable.

**4. Decompose Blocks by concern**

Never migrate a Prefect Block wholesale into "just an env var". Split it by purpose:

- **secret data** → ZenML secrets
- **cloud/service credentials** → service connectors
- **infrastructure config** → stack/orchestrator settings
- **runtime config** → YAML or pipeline parameters

**5. Keep migration comments short and explicit**

Use:
- `# Migration note:` for brief caveats
- `# TODO(migration):` for unsupported or manual-attention items

#### Handling approximate translations

When an approximation is safe enough to generate, add a short inline comment:

```python
@step
def load_secret(secret_name: str) -> str:
    # Migration note: this was a Prefect Secret block. ZenML stores secrets
    # separately from runtime config, so the lookup path and lifecycle differ.
    ...
```

#### Handling absent patterns

For patterns with no trustworthy ZenML equivalent:

1. add a `# TODO(migration):` comment,
2. record it in `MIGRATION_REPORT.md`,
3. suggest a redesign.

```python
# TODO(migration): UNSUPPORTED — original Prefect flow used allow_failure().
# ZenML does not provide dependency-level failure tolerance with the same
# semantics. Redesign this edge using an explicit result envelope artifact.
```

### Phase 4: Produce the Migration Report

After generating the ZenML project, produce a `MIGRATION_REPORT.md` in the project root.

```markdown
# Migration Report: [Prefect Flow] → [ZenML Pipeline]

## Summary
- **Source**: Prefect flow `[flow_name]`
- **Target**: ZenML pipeline `[pipeline_name]`
- **Components migrated**: X direct, Y approximate, Z flagged

## Direct Translations
| Prefect Pattern | ZenML Equivalent | Notes |
|---|---|---|
| `@task` `extract_data` | `steps/extract_data.py` | Clean task→step translation |

## Approximate Translations
| Prefect Pattern | ZenML Equivalent | What Changed |
|---|---|---|
| Deployment schedule | `Schedule(...)` | Scheduling support depends on orchestrator |
| Secret Block | ZenML secret | Config lives in a different system |

## Flagged for Review
| Prefect Pattern | Severity | Issue | Suggested Redesign |
|---|---|---|---|
| `allow_failure()` | HIGH | No direct ZenML equivalent | Return explicit success/error artifact |
| `pause_flow_run()` | HIGH | No drop-in pause/suspend state model | Use explicit approval/wait workflow |

## Execution Model Changes
- Was the original Prefect flow dynamic at runtime?
- Did the migration stay static, or require `@pipeline(dynamic=True)`?
- What behavior changed because ZenML compiles the DAG differently?

## State / Failure Handling Changes
- Which State-based patterns were removed or redesigned?
- Were failures turned into explicit data artifacts?

## Configuration and Deployment Mapping
- Which Blocks became secrets?
- Which became YAML config?
- Which deployment/work-pool settings now live in orchestrator or stack config?

## What's NOT Migrated
[List stateful control-plane behavior, transactions, Cloud-only features, or other unsupported patterns.]

## What You Get for Free After Migration
- Artifact versioning and lineage
- Step caching
- Stack portability
- Service connectors
- Model Control Plane (where relevant)

## Recommended Next Steps
1. Run `zenml-quick-wins`
2. Install the ZenML docs MCP server
3. Review each flagged redesign item
4. Use `zenml-pipeline-authoring` for Docker, YAML, custom materializers, or deployment details
```

### Phase 5: Suggest Next Steps

After migration is complete, always communicate the next steps clearly.

#### 1. Run the `zenml-quick-wins` skill

This should almost always be the next step:

> "Now that the migration is done, I'd recommend running the `zenml-quick-wins` skill to add metadata logging, experiment tracking, alerts, and other production features."

#### 2. Include documentation links for flagged patterns

For flagged items, link to the most relevant ZenML docs. Common links:

- Execution model: `https://docs.zenml.io/concepts/steps_and_pipelines/execution`
- Dynamic pipelines: `https://docs.zenml.io/concepts/steps_and_pipelines/dynamic_pipelines`
- Scheduling: `https://docs.zenml.io/concepts/steps_and_pipelines/scheduling`
- Service connectors: `https://docs.zenml.io/concepts/service_connectors`
- Secrets: `https://docs.zenml.io/concepts/secrets`

#### 3. Suggest installing the ZenML docs MCP server

> "For easier access to ZenML documentation while you work, you can install the ZenML docs MCP server: `claude mcp add zenmldocs --transport http https://docs.zenml.io/~gitbook/mcp`"

#### 4. Offer community support for hard migration gaps

When there are 2+ HIGH-severity flags, generate a ready-to-post Slack message for `zenml.io/slack` that includes:

- what is being migrated,
- the unsupported Prefect patterns,
- the redesigns already attempted,
- and a clear ask for suggestions.

Use this template:

```markdown
**Prefect → ZenML Migration Help**

I'm migrating a Prefect workflow that uses [patterns]. The migration skill flagged these as needing redesign:

1. **[Pattern]**: [brief description + small code snippet]
   - Suggested workaround: [X]
   - Why this matters: [what behavior would change]

2. **[Pattern]**: [brief description + small code snippet]
   - Suggested workaround: [Y]

I'm looking for advice on whether there's a better ZenML pattern, a feature I'm missing, or an upcoming capability that would make this migration cleaner.
```

#### 5. Offer GitHub issues for genuine feature gaps

If the migration exposes a real ZenML capability gap — not just "works differently", but a reusable missing feature — offer to open an issue on `zenml-io/zenml`.

#### 6. Suggest running `/simplify`

Migration often leaves verbose comments and slightly mechanical structure behind. Always suggest `/simplify` once the migration is functionally complete.

#### 7. Recommend `zenml-pipeline-authoring` for deeper follow-up work

Use `zenml-pipeline-authoring` for:
- Docker settings
- YAML config
- custom materializers
- pipeline deployment details

## Important Behavioral Differences to Communicate

### Dynamic Prefect execution ≠ static ZenML execution

In Prefect, flow code can make orchestration decisions while the run is already happening. In ZenML static pipelines, the DAG is compiled first. That is the single most important migration difference.

### Prefect State objects ≠ ZenML run/step status

Prefect lets workflow code inspect and route on state objects. ZenML records run and step status, but the authoring model is not "pass around State objects and branch on them."

### Prefect results ≠ ZenML artifacts

Prefect results can be optionally persisted and configured with storage/serializers. ZenML step outputs are first-class, versioned artifacts by default.

### Blocks ≠ one ZenML object

Prefect Blocks combine multiple concerns. ZenML splits them across secrets, connectors, stack components, YAML config, and parameters.

### Prefect Deployments ≠ ZenML pipeline deployments

Prefect Deployments are batch-run configuration. ZenML pipeline deployments are long-running HTTP services. For scheduled batch runs, the closer ZenML concepts are usually schedules, orchestrators, and sometimes snapshots — not HTTP deployments.

## Anti-Patterns in Migration

| Anti-pattern | Why it is wrong | What to do instead |
|---|---|---|
| Replacing `allow_failure()` with a global continue-on-failure mode | Changes dependency-level failure semantics | Redesign with explicit success/error artifacts |
| Translating runtime branches into static `if` statements on step outputs | Static pipelines cannot branch on artifact values | Use dynamic pipelines or redesign |
| Turning all Blocks into environment variables | Loses schema, discoverability, and concern separation | Split into secrets, connectors, stack config, YAML |
| Treating Prefect Deployments as ZenML HTTP deployments | They solve different problems | Map scheduled batch execution to schedules/orchestrators |
| Assuming Dask/Ray task-runner behavior survives automatically | Concurrency and isolation models differ | Re-evaluate infra and step boundaries explicitly |
| Silently dropping `cache_key_fn` logic | Can change business semantics, not just performance | Flag and redesign caching explicitly |

## References

### Detailed reference files

- [references/concept-map.md](references/concept-map.md) — Full concept mapping tables for Prefect primitives, states, concurrency, Blocks, deployments, and automation patterns
- [references/code-patterns.md](references/code-patterns.md) — Side-by-side translations for common Prefect patterns and redesign examples for unsafe ones
- [references/gaps-and-flags.md](references/gaps-and-flags.md) — Must-flag patterns, behavioral differences, decision tree, and migration anti-patterns

### ZenML documentation

For topics beyond migration, query the ZenML docs at https://docs.zenml.io.
