---
name: flyte-to-zenml-migration
description: >-
  Migrate Flyte workflows, tasks, LaunchPlans, and Flytekit code to idiomatic
  ZenML pipelines. Handles concept mapping (`@task`->`@step`,
  `@workflow`->`@pipeline`, `map_task()`->dynamic `.map()`,
  `conditional()`->dynamic branching, `LaunchPlan`->schedule/config split),
  code translation, special-type migration (`FlyteFile`, `FlyteDirectory`,
  `StructuredDataset`, `FlyteSchema`), Docker/image mapping, and flags
  unsupported patterns (`@eager`, `ContainerTask`, reference entities,
  checkpointing, interruptible semantics) for human review. Use this skill
  whenever the user mentions Flyte migration, converting Flyte to ZenML,
  porting Flyte workflows, replacing Flyte with ZenML, or asks how a Flyte
  concept maps to ZenML -- even if they do not explicitly say "migrate". Also
  use when they paste Flytekit code and ask to make it work with ZenML, or when
  they describe a workflow using Flyte terminology (`@dynamic`, `LaunchPlan`,
  `map_task`, `conditional`, `ImageSpec`, `FlyteFile`, `StructuredDataset`,
  `reference_task`, `reference_workflow`) in a ZenML context. If the user just
  asks a quick conceptual question ("what is the ZenML equivalent of
  LaunchPlan?" or "how should FlyteFile map?"), answer it directly from the
  concept map -- no need to run the full migration workflow.
---

# Migrate Flyte to ZenML

This skill translates Flyte workflows into idiomatic ZenML pipelines. It handles the full migration workflow: analyzing Flytekit code, classifying each concept, translating what maps cleanly, flagging what needs redesign, and producing a working ZenML project with a migration report.

## How migration works at a high level

Flyte and ZenML are closer to each other than Flyte and older scheduler-first systems like Airflow. Both are Python-first orchestration frameworks that care about typed execution units, retries, scheduling, and containerized execution. So the migration is often more like “rewire the execution story” than “translate operator objects into functions.”

But there are still sharp edges. Flyte has a richer workflow transport type system (`FlyteFile`, `StructuredDataset`, `FlyteSchema`), a stronger registered execution surface via `LaunchPlan`, and special execution features like `@dynamic`, `@eager`, `map_task()`, `conditional()`, `ContainerTask`, and reference entities. ZenML can often reach the same business outcome, but not always with the same runtime semantics.

This means migration is not a decorator-swap exercise. Some patterns translate directly, some need approximation, and some require honest redesign.

### The three mapping types

Every Flyte concept falls into one of these categories:

| Type | Meaning | Action |
|------|---------|--------|
| **Direct** | Clean 1:1 mapping exists | Translate automatically |
| **Approximate** | Conceptual equivalent exists but semantics differ | Translate with caveats noted in the migration report |
| **Absent** | No safe ZenML core equivalent | Flag for human review with redesign suggestions |

See [references/concept-map.md](references/concept-map.md) for the full mapping tables.

## The Migration Workflow

### Phase 1: Receive and Analyze the Flyte Code

Ask the user for the Flyte source before doing anything else. The ideal input set is:

- Flyte task and workflow Python files
- `LaunchPlan` definitions
- `ImageSpec` or other container/image configuration
- plugin-backed task config modules (`task_config=...`, plugin resources, external job specs)
- any custom types, materializers, or serializers
- any Union-specific code if present

Read everything thoroughly. For each workflow, identify:

1. **Execution units** -- plain `@task`, `@workflow`, nested workflows, subworkflows
2. **Type system** -- primitives, collections, dataclasses, Pydantic models, `FlyteFile`, `FlyteDirectory`, `StructuredDataset`, `FlyteSchema`
3. **Control flow** -- `@dynamic`, `map_task()`, `conditional()`, `@eager`
4. **Execution metadata** -- retries, `cache=True`, `cache_version`, `timeout`, `interruptible=True`
5. **Infrastructure** -- `Resources`, `ImageSpec`, per-task images, `ContainerTask`, plugin `task_config`
6. **Scheduling and trigger surface** -- `LaunchPlan`, `default_inputs`, `fixed_inputs`, schedules, notifications
7. **Cross-project boundaries** -- `reference_task`, `reference_workflow`, `reference_launch_plan`
8. **Advanced recovery/reporting** -- checkpointing, Decks, custom HTML/report artifacts
9. **Union-only extensions** -- Actors, reusable containers, Union Artifacts, Union Channels, commercial control-plane features

### Phase 2: Classify and Plan

For each component identified in Phase 1, classify it as direct, approximate, or absent. Use the quick guide below and the full mapping tables in [references/concept-map.md](references/concept-map.md).

#### Quick classification guide

Use **direct** sparingly. In Flyte migrations, many things are mechanically easy to rewrite but still semantically different enough that they belong in the **approximate** bucket.

**Usually straightforward translations (often still classified as approximate):**
- plain `@task` -> `@step`
- plain `@workflow` -> `@pipeline`
- primitive/container types where ZenML built-in materializers are enough
- basic retries -> `StepRetryConfig`
- simple resource hints -> `ResourceSettings`

**Approximate translations (translate with caveats):**
- `@dynamic` -> `@pipeline(dynamic=True)`
- `map_task()` -> `.map()` inside a dynamic pipeline
- `conditional()` -> dynamic pipeline branching with `.load()`
- `StructuredDataset` / `FlyteSchema` -> dataframe or table artifact
- `FlyteFile` / `FlyteDirectory` -> `Path` artifact or wrapper type + materializer
- `ImageSpec` / per-task image settings -> `DockerSettings`
- basic `LaunchPlan` scheduling/defaults -> pipeline defaults + `Schedule`
- notifications -> hooks / alerters / external alerts
- Decks -> metadata logging, visualizers, or explicit report artifacts

When in doubt, classify Flyte decorator-level concepts as **approximate** and explain the runtime difference in the migration report.

**Absent / needs redesign (flag for human review):**
- `@eager`
- `ContainerTask`
- `reference_task`, `reference_workflow`, `reference_launch_plan`
- `interruptible=True`
- portable timeout parity
- intra-task checkpointing
- `map_task(min_success_ratio=...)`
- Union Actors / reusable container semantics
- Union Artifacts / Union Channels when semantics are unclear

#### Present the migration plan

Before writing code, present a concrete summary:

> "Here's what I found in your Flyte code:
> - **Direct translations** (will migrate cleanly): [list]
> - **Approximate translations** (will work but with noted caveats): [list]
> - **Needs redesign** (cannot auto-migrate safely): [list with brief explanation]
>
> Shall I proceed with the migration?"

If there are HIGH-severity flags, explain them in story form: what the Flyte code was relying on, what ZenML does differently, and what redesign is safest.

### Phase 3: Generate ZenML Code

Translate the Flyte workflow into a ZenML project. Follow these conventions strictly.

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
├── materializers/            # Only when special Flyte types need them
├── configs/
│   ├── dev.yaml
│   └── prod.yaml
├── run.py                    # CLI entry point (argparse, not click)
├── README.md
└── pyproject.toml
```

This matches the `zenml-pipeline-authoring` skill's conventions. Key rules:
- one step per file in `steps/`
- separate pipeline definition from execution
- `run.py` uses `argparse`
- `pyproject.toml` uses `zenml>=0.94.1` and `requires-python = ">=3.12"`
- always generate `configs/dev.yaml` and `configs/prod.yaml`
- always generate `README.md` explaining the migrated pipeline and what still needs human review
- run `zenml init` at project root

#### Translation rules

See [references/code-patterns.md](references/code-patterns.md) for detailed side-by-side examples. The core rules are:

- `@task` -> `@step`
- `@workflow` -> `@pipeline`
- `@dynamic` -> `@pipeline(dynamic=True)`
- `map_task()` -> `.map()` only inside dynamic pipelines
- `conditional()` -> `.load()`-driven branching in dynamic pipelines
- `Resources(...)` -> `ResourceSettings(...)`
- `ImageSpec(...)` -> `DockerSettings(...)`
- `retries=N` -> `StepRetryConfig(max_retries=N, ...)`
- LaunchPlan schedule -> `Schedule(...)`
- LaunchPlan defaults/fixed inputs -> explicit pipeline defaults, wrapper pipeline, or deployment-specific config
- LaunchPlan notifications -> hooks / alerter hooks

#### How to treat Flyte special types

These choices should be deliberate, not improvised:

**`FlyteFile` / `FlyteDirectory`**
- Default target: `Path` artifact
- If the original code depended on remote URI, provenance, or lazy localization semantics, do **not** flatten it to `str`
- Use a wrapper type + custom materializer, or preserve the missing semantics as artifact metadata

**`StructuredDataset` / `FlyteSchema`**
- Default target: `pd.DataFrame`, `polars.DataFrame`, or `pyarrow.Table`
- Match the concrete data shape the code actually uses
- If schema, format, or backend metadata mattered in Flyte, preserve it with:
  - a validator step
  - explicit metadata logging
  - or a custom materializer

**Externally managed data**
- If the Flyte workflow consumed data that was not produced inside the current run, prefer `register_artifact` / `ExternalArtifact`
- Do not pretend an external file was created by a ZenML step just to make the code look tidy

**Exotic Python objects**
- A custom materializer is the preferred destination
- `CloudpickleMaterializer` can be a temporary unblocker, but never present it as the intended long-term production state

#### Code comment style

Keep migration comments brief and useful:

- use `# Migration note:` for short caveats
- use `# TODO(migration):` for items requiring user action
- put detailed reasoning in `MIGRATION_REPORT.md`, not in large inline comments

#### Handling approximate translations

When translating an approximate pattern, add a short inline note that explains the semantic difference:

```python
@step
def read_remote_input(path: Path) -> pd.DataFrame:
    # Migration note: the original Flyte workflow used StructuredDataset
    # metadata to control reader behavior. This ZenML step now assumes the
    # artifact is a pandas DataFrame and validates the expected columns here.
    ...
```

#### Handling absent patterns

For patterns with no safe ZenML equivalent:

1. add a clearly marked `# TODO(migration)` comment
2. include the item in the migration report
3. suggest a redesign approach instead of silently approximating

```python
# TODO(migration): UNSUPPORTED -- Flyte ContainerTask with file-based IO
# contract. ZenML has no first-class raw container task primitive here.
# Redesign this as a wrapper step that submits the external job and stages IO
# explicitly, or replace it with a custom step operator.
@step
def run_external_job(...) -> None:
    ...
```

### Phase 4: Produce the Migration Report

After generating the ZenML project, produce a `MIGRATION_REPORT.md` in the project root. Use this structure:

```markdown
# Migration Report: [workflow] -> [pipeline]

## Summary
- **Source workflow**: `[workflow_name]`
- **Target pipeline**: `[pipeline_name]`
- **Tasks migrated**: X direct, Y approximate, Z flagged
- **LaunchPlans reviewed**: N
- **Plugin-backed tasks reviewed**: M

## Direct Translations
| Flyte Concept | ZenML Target | Notes |

## Approximate Translations
| Flyte Concept | ZenML Target | What Changed |

## Flagged for Review
| Flyte Pattern | Severity | Issue | Suggested Redesign |

## Type and Artifact Mapping
| Flyte type/pattern | ZenML representation | Notes |

## Scheduling and LaunchPlan Mapping

## Infrastructure and Containerization Mapping

## Limitations and Key Differences

## What's NOT Migrated

## What You Get for Free After Migration

## Recommended Next Steps
```

In the report, put **Limitations and Key Differences** before **What You Get for Free After Migration** so the user sees the caveats first.

### Phase 5: Suggest Next Steps

After migration is complete, always include a "Recommended Next Steps" section in the report AND communicate it to the user.

#### 1. Run the `zenml-quick-wins` skill

Always suggest this first:

> "Now that the migration is done, I'd recommend running the `zenml-quick-wins` skill to add metadata logging, experiment tracking, alerts, and other production-readiness features."

#### 2. Documentation links for flagged patterns

For every flagged pattern, include the relevant ZenML docs. Common Flyte-migration links:

- Dynamic pipelines: `https://docs.zenml.io/how-to/steps-pipelines/dynamic-pipelines`
- Scheduling: `https://docs.zenml.io/how-to/steps-pipelines/schedule-a-pipeline`
- Orchestrators: `https://docs.zenml.io/stacks/stack-components/orchestrators`
- Containerization: `https://docs.zenml.io/how-to/containerization/containerization`
- Service connectors / auth: `https://docs.zenml.io/how-to/infrastructure-deployment/auth-management`
- Materializers: `https://docs.zenml.io/concepts/artifacts/materializers`
- Deployment: `https://docs.zenml.io/how-to/deployment/deployment`

#### 3. Suggest installing the ZenML docs MCP server

> "For easier access to ZenML docs while you finish the migration, you can install the ZenML docs MCP server: `claude mcp add zenmldocs --transport http https://docs.zenml.io/~gitbook/mcp`"

#### 4. Community support for unsupported patterns

When there are **2+ HIGH-severity flags**, generate a copy-paste Slack message for `zenml.io/slack` that includes:
- what is being migrated
- the unsupported Flyte patterns
- a short code snippet for each
- the proposed workaround
- a clear ask for better patterns or upcoming support

```markdown
**Flyte -> ZenML Migration Help**

I'm migrating a Flyte workflow (`[workflow_name]`) that uses [patterns]. The migration skill flagged these as needing redesign:

1. **[Pattern]**: [brief description + code snippet]
   - Suggested workaround: [X]
   - Why this matters: [what changes without a proper solution]

2. **[Pattern]**: [brief description + code snippet]
   - Suggested workaround: [Y]

I've implemented the workarounds above, but I'm wondering if there's a better approach, an upcoming feature, or a pattern I'm missing.
```

#### 5. Open GitHub issues for genuine feature gaps

When the migration reveals a real capability gap in ZenML, offer to open a GitHub issue on `zenml-io/zenml` using `gh issue create`.

#### 6. Run `/simplify`

After migration is complete, always suggest running `/simplify` on the generated code. Migration often leaves temporary comments, repeated wrappers, and verbose explanations that should be cleaned up.

#### 7. Use `zenml-pipeline-authoring` for deeper customization

Recommend `zenml-pipeline-authoring` for:
- Docker settings for remote execution
- YAML configuration for multiple environments
- custom materializers for Flyte-like special types
- deployment and serving patterns

## Important Behavioral Differences to Communicate

Always mention the relevant ones in the migration report.

### Flyte transport types != ZenML artifacts

Flyte's special types are part of the transport layer. ZenML relies on Python types plus materializers. That changes:
- **file semantics** -- `FlyteFile` / `FlyteDirectory` often need more than a plain string path
- **tabular semantics** -- `StructuredDataset` and `FlyteSchema` may carry metadata that has to be recreated intentionally
- **serialization** -- ZenML makes the materialization strategy explicit
- **external data** -- `ExternalArtifact` / `register_artifact` are often the cleanest migration target

### Dynamic execution semantics

Flyte dynamic features are runtime engine features. ZenML dynamic pipelines load values back into Python to shape the graph. That means:
- `@dynamic` is only an approximate match
- `map_task()` and `.map()` are similar in goal but not identical in backend semantics
- `conditional()` must be treated carefully when it depends on runtime values

### LaunchPlan != Schedule

`LaunchPlan` is not just Flyte's cron object. It is also the registered execution surface with defaults, fixed inputs, and notifications. ZenML's `Schedule` only covers the scheduling slice. The rest has to be made explicit with pipeline defaults, config, deployments, or wrapper pipelines.

### Infrastructure differences

Flyte makes raw container execution and plugin-backed task contracts first-class. ZenML is stronger at portable Python pipelines plus stack abstractions. This means:
- `ImageSpec` and `DockerSettings` are close in spirit, not identical in lifecycle
- `ContainerTask` is a redesign boundary
- retries map better than timeouts or interruptible semantics

## Anti-Patterns in Migration

| Anti-pattern | Why it's wrong | What to do instead |
|---|---|---|
| Translating `FlyteFile` to `str` | Loses artifact semantics and remote/localization behavior | Use `Path`, plus metadata or a wrapper type if URI semantics matter |
| Translating `StructuredDataset` to `Any` | Destroys the tabular contract and hides schema drift | Use explicit dataframe/table types |
| Replacing `map_task()` with a plain Python loop in a static pipeline | Removes parallel semantics and per-item behavior | Use a dynamic pipeline with `.map()` or redesign |
| Replacing `conditional()` with plain `if` in a non-dynamic pipeline | Breaks runtime branch semantics | Use `@pipeline(dynamic=True)` or move the decision into a step |
| Mapping `LaunchPlan` to only `Schedule` | Loses fixed inputs, defaults, notifications, and trigger identity | Split it into config, wrapper pipelines, deployments, and schedules |
| Treating `interruptible=True` as "just add retries" | Spot/preemptible behavior is not the same as retry behavior | Move spot handling to the target backend config |
| Rewriting `ContainerTask` as a normal Python step without reviewing IO protocol | Changes the execution contract completely | Wrap the external job or build a custom operator |
| Ending the migration on `CloudpickleMaterializer` | It is a temporary escape hatch, not a stable design | Create proper materializers or simplify the data contract |
| Mirroring Flyte plugin config fields 1:1 | Plugin semantics do not carry over automatically | Migrate the business outcome, not the config object |

## References

### Detailed reference files

- [references/concept-map.md](references/concept-map.md) -- Full concept mapping tables for Flyte concepts, special types, LaunchPlans, plugins, and Union features
- [references/code-patterns.md](references/code-patterns.md) -- Side-by-side Flyte -> ZenML code translations for workflows, dynamic execution, special types, LaunchPlans, image settings, retries, and raw container patterns
- [references/gaps-and-flags.md](references/gaps-and-flags.md) -- Must-flag patterns, behavioral differences, migration decision tree, and the full list of "do not silently approximate" patterns

### ZenML documentation

For topics beyond migration (stack setup, artifact handling, deployment, productionization), query the ZenML docs at https://docs.zenml.io.
