---
name: metaflow-to-zenml-migration
description: >-
  Migrate Metaflow flows and Outerbounds-flavored Metaflow projects to
  idiomatic ZenML pipelines. Handles concept mapping (FlowSpec->pipeline,
  @step->@step, self.* artifacts->explicit returns and inputs), code
  translation for Parameters, IncludeFile, Config, self.next transitions,
  branch/join, foreach, scheduling, retry/resource/dependency decorators, and
  flags unsupported or high-risk patterns (@catch, merge_artifacts, resume and
  checkpoint semantics, recursion, event triggers, @batch) for human review.
  Use this skill whenever the user mentions Metaflow migration, converting
  FlowSpec code, porting flows from Metaflow or Outerbounds, replacing
  Metaflow orchestration with ZenML, or asks how a Metaflow concept maps to
  ZenML -- even if they don't explicitly say "migrate". Also use when they
  paste FlowSpec code or describe workflows using Metaflow terminology
  (self.next, foreach, current, Parameter, IncludeFile, Config, @catch,
  @kubernetes, @batch, Runner, Deployer) in a ZenML context. If the user just
  asks a quick conceptual question ("what's the ZenML equivalent of
  merge_artifacts?"), answer it directly from the concept map -- no need to run
  the full migration workflow.
---

# Migrate Metaflow to ZenML

This skill translates Metaflow flows into idiomatic ZenML pipelines. It handles the full migration workflow: analyzing `FlowSpec` code, classifying each pattern, translating what maps cleanly, flagging what needs redesign, and producing a working ZenML project.

## How migration works at a high level

Metaflow and ZenML are deceptively close cousins. Both talk about steps, artifacts, local vs remote execution, and moving the same code between environments. But they tell that story in different ways:

- **Metaflow** builds a workflow around a `FlowSpec` class, `@step` methods, `self.next(...)` transitions, and `self.*` assignments that become persisted artifacts.
- **ZenML** builds a workflow around a `@pipeline` function, standalone `@step` functions, and explicit step inputs and outputs that become typed, versioned artifacts.

So this is not a rename-the-primitives migration. The dangerous cases are the ones that still "look right" after a naive rewrite but silently change behavior: join semantics, `foreach`, `merge_artifacts`, `@catch`, resume/checkpoint behavior, conditional branching, recursion, and platform-specific decorators like `@batch`.

### The three mapping types

Every Metaflow concept falls into one of these categories:

| Type | Meaning | Action |
|------|---------|--------|
| **Direct** | Clean 1:1 mapping exists | Translate automatically |
| **Approximate** | Conceptual equivalent exists but semantics differ | Translate with caveats noted in the migration report |
| **Absent** | No safe ZenML equivalent exists | Flag for human review with redesign suggestions |

See [references/concept-map.md](references/concept-map.md) for the full mapping tables.

## The Migration Workflow

### Phase 1: Receive and Analyze the Metaflow Code

Ask the user for their Metaflow flow files, supporting modules, configuration files, and any deployment/runtime commands they currently use. Read everything before writing code.

For each flow, identify:

1. **Flow structure**
   - `FlowSpec` class name
   - `start` and `end` steps
   - every `self.next(...)` transition
   - whether transitions are linear, branching, conditional, recursive, or `foreach`
2. **Artifact flow**
   - every `self.<name> = ...` assignment
   - where each artifact is read later
   - whether joins depend on implicit propagation or `merge_artifacts(inputs)`
3. **Control flow**
   - linear chains
   - branch fan-out and joins
   - `foreach`, `self.input`, `self.index`
   - conditional branching
   - recursion or re-entry patterns
4. **Parameters and external inputs**
   - `Parameter`
   - `IncludeFile`
   - `Config`
   - CLI-time or deployment-time overrides
5. **Decorators**
   - `@retry`
   - `@catch`
   - `@timeout`
   - `@resources`
   - `@batch`
   - `@kubernetes`
   - `@conda`, `@pypi`, `@conda_base`
   - `@environment`
   - `@secrets`
   - `@card`
   - `@schedule`
   - `@trigger`, `@trigger_on_finish`
   - `@project`
   - `@checkpoint`
   - custom decorators or `--with <decorator>` overlays
6. **Runtime and platform features**
   - `current`
   - `metaflow.client`
   - `Runner`
   - `Deployer`
   - `resume`
   - `metaflow.S3`
   - namespaces and tags
7. **Outerbounds features**
   - Fast Bakery / dependency baking
   - `@docker`
   - `@gpu_profile`
   - project assets
   - deployment endpoints

If the user gives you only a quick conceptual question, answer from the concept map and stop there. Use the full migration workflow only when there is real code or a real migration design problem to solve.

### Phase 2: Classify and Plan

For each pattern from Phase 1, classify it as direct, approximate, or absent. Use the quick guide below plus the detailed tables in [references/concept-map.md](references/concept-map.md) and [references/gaps-and-flags.md](references/gaps-and-flags.md).

#### Quick classification guide

**Direct translations (translate automatically):**
- linear `self.next(self.a)` chains
- simple `@step` method logic -> ZenML `@step`
- simple `Parameter` values -> pipeline parameters
- `@retry` -> `StepRetryConfig`

**Approximate translations (translate with caveats):**
- `FlowSpec` -> `@pipeline`
- `self.*` artifacts -> explicit step returns and downstream inputs
- branching + join -> explicit reducer/join steps
- `foreach` -> `@pipeline(dynamic=True)` plus `.map()` and explicit reducer/join steps; manual loops may also need `.load()` for decisions and `.chunk(idx)` for DAG wiring
- `@resources` -> `ResourceSettings`
- `@kubernetes` -> Kubernetes orchestrator or step operator settings
- `@conda` / `@pypi` / Fast Bakery -> `DockerSettings` and container-image design
- `@schedule` -> OSS/orchestrator-backed `Schedule(...)`, with target orchestrator support, singular `zenml pipeline schedule ...` lifecycle commands where supported, and cron semantics called out explicitly; ZenML Pro schedule triggers are separate snapshot trigger objects
- dynamic-pipeline-heavy flows -> only treat as a realistic target when the chosen orchestrator is one of ZenML's documented dynamic-pipeline backends (`local`, `local_docker`, `kubernetes`, `sagemaker`, `vertex`, `azureml`); dynamic pipelines default to `STOP_ON_FAILURE`, support `FAIL_FAST` with caveats, and do not support `CONTINUE_ON_FAILURE`
- `Config` -> YAML config / `.with_options(config_file=...)`
- `current` -> `get_step_context()` for narrow step/run metadata lookup only; broader `current.*` usage must be flagged
- `metaflow.client` -> `zenml.client.Client` only for limited lineage/artifact lookup; richer history traversal should be flagged
- Runner / Deployer flows -> snapshots, deployments, SDK or API-triggered runs; use ZenML Pro schedule/platform-event triggers attached to snapshots only when their supported trigger semantics fit the source behavior

**Absent / must flag for review:**
- `@catch`
- `merge_artifacts`
- direct recursion as a workflow primitive
- exact `resume` semantics
- `@checkpoint`
- `@batch` as a direct portable equivalent
- portable `@timeout` semantics
- `@trigger` / `@trigger_on_finish`; ZenML Pro platform-event triggers may fit supported ZenML platform lifecycle events, but Metaflow trigger semantics are not a direct 1:1 migration
- business logic that depends on rich `current.*` state
- Outerbounds-only features with no clear ZenML surface

#### Present the plan before coding

Before writing migration code, summarize the flow like this:

> "Here's what I found in your Metaflow flow:
> - **Direct translations** (will migrate cleanly): [list]
> - **Approximate translations** (will work but with caveats): [list]
> - **Needs redesign** (cannot be auto-migrated safely): [list with explanation]
>
> Shall I proceed with the migration?"

If there are HIGH-severity flags, explain them concretely in story form: what the Metaflow flow currently does, where the behavior lives, why ZenML cannot preserve it directly, and what redesign path is most honest.

### Phase 3: Generate ZenML Code

Translate the Metaflow flow into a ZenML project. Follow these conventions strictly.

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
├── materializers/            # Custom materializers if needed
├── configs/
│   ├── dev.yaml
│   └── prod.yaml
├── run.py                    # CLI entry point (argparse, not click)
├── README.md
└── pyproject.toml
```

Key rules:

- one step per file in `steps/`
- separate pipeline definition from execution
- `run.py` uses `argparse`
- `pyproject.toml` should use `requires-python = ">=3.12"` and a current ZenML dependency appropriate for the target environment
- always generate `configs/dev.yaml` and `configs/prod.yaml`
- always generate a `README.md` that explains what changed, how to run, and what still needs manual attention
- include a brief ASCII DAG diagram in the pipeline module docstring
- run `zenml init` at the project root

#### Translation patterns

For each Metaflow step, apply the right translation. See [references/code-patterns.md](references/code-patterns.md) for side-by-side examples.

**Core rule:** move step logic out of the `FlowSpec` class and into standalone `@step` functions. Replace implicit `self.*` state with explicit function returns and typed inputs.

```python
# Metaflow
class MyFlow(FlowSpec):
    @step
    def start(self):
        self.x = 1
        self.next(self.end)

# ZenML
@step
def start() -> int:
    return 1

@step
def end(x: int) -> None:
    print(x)

@pipeline
def my_pipeline() -> None:
    x = start()
    end(x)
```

**`self.*` artifacts -> explicit artifacts**:

```python
# Metaflow
self.features = build_features(self.raw)

# ZenML
@step
def build_features_step(raw: list[int]) -> list[int]:
    return build_features(raw)
```

**Parameters -> pipeline parameters**:

```python
# Metaflow
class TrainFlow(FlowSpec):
    alpha = Parameter("alpha", default=0.1)

# ZenML
@pipeline
def train_pipeline(alpha: float = 0.1) -> None:
    ...
```

**Retries -> `StepRetryConfig`**:

```python
@step(retry=StepRetryConfig(max_retries=3, delay=60, backoff=2))
def flaky_step() -> None:
    ...
```

**Scheduling -> `Schedule(...)`**:

```python
from zenml.config.schedule import Schedule

schedule = Schedule(cron_expression="0 2 * * *")
my_pipeline.with_options(schedule=schedule)()
```

Always note that scheduling support depends on the orchestrator. In OSS, a `Schedule(...)` is attached to the pipeline run and managed with singular `zenml pipeline schedule ...` commands where supported. In ZenML Pro, schedule triggers are server-side trigger objects attached to snapshots (`zenml trigger schedule create`, `attach`, `list`, `delete`). Metaflow `@schedule`, `@trigger`, and `@trigger_on_finish` semantics still need explicit review rather than a direct rename. Check the scheduling table in [references/concept-map.md](references/concept-map.md).

#### Handling approximate translations

When a pattern is close but not identical, keep the generated code honest with short inline comments:

```python
@step
def join_results(left_score: float, right_score: float) -> float:
    # Migration note: Metaflow join steps can rely on implicit artifact
    # propagation. ZenML requires the join contract to be explicit, so all
    # branch outputs needed downstream are listed here directly.
    return max(left_score, right_score)
```

Approximation comments should be short and actionable. Put the long explanation in the migration report, not in the code.

#### Handling absent patterns

Never silently approximate absent patterns. Instead:

1. add a `# TODO(migration):` comment in the generated code
2. record it in the migration report
3. suggest a redesign

```python
# TODO(migration): UNSUPPORTED -- original flow used @catch to convert step
# failure into a successful downstream continuation path. ZenML has no direct
# equivalent. Consider returning an explicit Result/Error envelope from the step
# or splitting the recovery logic into a separate pipeline.
@step
def recovery_wrapper(...) -> ...:
    ...
```

### Phase 4: Produce the Migration Report

After generating the ZenML project, produce a `MIGRATION_REPORT.md` in the project root:

```markdown
# Migration Report: [Metaflow Flow] -> [ZenML Pipeline]

## Summary
- **Source**: Metaflow flow `[FlowSpec name]`
- **Target**: ZenML pipeline `[pipeline_name]`
- **Steps migrated**: X direct, Y approximate, Z flagged

## Direct Translations
| Metaflow Pattern | ZenML Equivalent | Notes |
|---|---|---|
| `@retry` on `train` | `StepRetryConfig` | Clean translation |

## Approximate Translations
| Metaflow Pattern | ZenML Equivalent | What Changed |
|---|---|---|
| `self.features` artifact propagation | explicit step outputs | downstream dependencies are now explicit |
| `foreach` fan-out | dynamic pipeline `.map()` | experimental and orchestrator-limited |

## Flagged for Review
| Metaflow Pattern | Severity | Issue | Suggested Redesign |
|---|---|---|---|
| `@catch` on `score_model` | HIGH | no direct placeholder-success behavior | return explicit error envelope |
| `merge_artifacts(inputs)` | HIGH | no implicit merge primitive | write explicit conflict resolution logic |

## Control-Flow Redesign Notes
[Explain branch/join, foreach, conditionals, or recursion changes.]

## Environment and Compute Mapping
[Explain dependency, Docker, step-operator, and resource changes.]

## Resume and Recovery Semantics
- **Original**: [How resume/checkpoint behaved in Metaflow]
- **Migrated**: [How caching/artifact reuse behaves in ZenML]
- **Important difference**: [Why this is approximate, not exact]

## What's NOT Migrated
[List unsupported decorators, platform features, or manual follow-ups.]

## What You Get for Free After Migration
- typed, versioned artifacts
- lineage and caching
- stack abstraction
- Model Control Plane
- service connectors
- pipeline deployments

## Recommended Next Steps
1. Run `zenml-quick-wins`
2. Install the ZenML docs MCP server
3. Review the flagged redesign items
4. Use `zenml-pipeline-authoring` for deeper customization
```

Always include the "Resume and Recovery Semantics" section when the source flow used `resume`, `@checkpoint`, `@catch`, or complex retry behavior.

### Phase 5: Suggest Next Steps

After migration, always include a next-steps section in the report and summarize it to the user.

#### 1. Run `zenml-quick-wins`

Always suggest this first:

> "Now that the migration is done, I'd recommend running the `zenml-quick-wins` skill to add metadata logging, experiment tracking, alerters, secrets, and other production features."

#### 2. Point to official ZenML docs for flagged patterns

Use current official ZenML docs when suggesting follow-up reading:

- Dynamic pipelines: `https://docs.zenml.io/how-to/steps-pipelines/dynamic-pipelines`
- Scheduling: `https://docs.zenml.io/how-to/steps-pipelines/scheduling`
- ZenML Pro triggers: `https://docs.zenml.io/getting-started/zenml-pro/triggers`
- Materializers: `https://docs.zenml.io/concepts/artifacts/materializers`
- Pipeline deployments: `https://docs.zenml.io/how-to/deployment/deployment`
- Service connectors: `https://docs.zenml.io/concepts/service_connectors`
- Stack components: `https://docs.zenml.io/concepts/stack_components`
- Models / Model Control Plane: `https://docs.zenml.io/concepts/models`

#### 3. Suggest the ZenML docs MCP server

> "For easier doc-grounded help while you work, you can install the ZenML docs MCP server: `claude mcp add zenmldocs --transport http https://docs.zenml.io/~gitbook/mcp`"

#### 4. Offer community help for real migration blockers

When there are **2 or more HIGH-severity flags**, generate a ready-to-send Slack message for `zenml.io/slack` that includes:

- what flow is being migrated
- which Metaflow features blocked a clean migration
- the workaround already attempted
- what the user wants help with

#### 5. Offer a GitHub issue for genuine feature gaps

If the migration surfaces a real missing ZenML capability, offer to open an issue on `zenml-io/zenml` with the blocked Metaflow pattern, the attempted workaround, and why the gap matters.

#### 6. Suggest `/simplify`

Always suggest running `/simplify` on the generated code after migration. Migration output often carries extra comments, duplicated plumbing, or defensive wrappers that can be cleaned up once the user has reviewed the semantics.

#### 7. Suggest `zenml-pipeline-authoring`

For deeper follow-up work, recommend `zenml-pipeline-authoring` for:

- Docker and container settings
- YAML configuration
- materializers
- step operators
- deployments and serving

## Important Behavioral Differences to Communicate

These are the places where users most easily get surprised after a migration.

### `self.*` artifacts != explicit step outputs

Metaflow lets a step quietly create many persisted artifacts just by assigning to `self.<name>`. ZenML persists what you explicitly return. If you forget to return something in ZenML, the downstream step will not magically find it later.

### Join semantics are explicit in ZenML

Metaflow joins can inherit artifacts implicitly and resolve ambiguity with `merge_artifacts(inputs)`. ZenML has no equivalent "carry forward whatever is unambiguous" rule. The join contract has to be written out by hand.

### Dynamic control flow is possible, but not the default

Metaflow can decide graph shape at step runtime with `self.next(...)`. ZenML static pipelines decide structure when the pipeline function runs. Runtime-dependent branching and fan-out generally require `@pipeline(dynamic=True)`. Dynamic pipelines are supported on `local`, `local_docker`, `kubernetes`, `sagemaker`, `vertex`, and `azureml`, but still carry important feature and runtime limitations: the default execution mode is `STOP_ON_FAILURE`, `FAIL_FAST` is supported with caveats around already-running inline steps, and `CONTINUE_ON_FAILURE` is unsupported.

### Resume is not caching

Metaflow `resume` works by step identity and prior run state. ZenML caching works by code, inputs, settings, and artifact lineage. They both help you avoid re-running work, but they are not semantic twins.

### Environment management shifts from decorator-driven installs to container design

Metaflow often expresses dependencies as decorators like `@conda`, `@pypi`, or Outerbounds baking workflows. ZenML expects you to think in terms of Docker images, stack components, and step runtime environments.

## Anti-Patterns in Migration

| Anti-pattern | Why it's wrong | What to do instead |
|---|---|---|
| Keeping a `FlowSpec` class and sprinkling ZenML decorators on methods | ZenML steps should be standalone callables with explicit inputs/outputs | Extract step logic into functions and rebuild the DAG in a `@pipeline` |
| Translating `self.*` to module-level mutable state | Loses artifact persistence and lineage | Return typed values from steps and pass them downstream explicitly |
| Silently replacing `merge_artifacts(inputs)` with "take one branch" | Changes join behavior | Write explicit merge/conflict logic and flag it |
| Rewriting `foreach` as a plain Python `for` loop without calling out the semantic change | Loses orchestrated fan-out, observability, and parallelism | Use dynamic pipelines where supported and document execution modes (`STOP_ON_FAILURE` default, `FAIL_FAST` caveats, no `CONTINUE_ON_FAILURE`), or flag the redesign |
| Pretending `@catch` is just `try/except` | Metaflow changes pipeline failure semantics | Return explicit error objects or redesign the failure boundary |
| Treating `resume` as identical to ZenML caching | They decide reuse differently | Explain the difference in the migration report |
| Mapping `@batch` directly to a generic remote stack | Hides real compute and orchestration differences | Flag as redesign and choose the target compute model explicitly |
| Assuming `current.*` metadata always has a ZenML twin | ZenML context is narrower | Replace with explicit inputs, metadata logging, or step context where possible |
| Copying Outerbounds deploy semantics decorator-for-decorator | The control plane is different | Treat deployment and serving as redesign work using ZenML deployments/model deployers |

## References

### Detailed reference files

- [references/concept-map.md](references/concept-map.md) -- full concept mapping tables for Metaflow, ZenML, and common Outerbounds extensions
- [references/code-patterns.md](references/code-patterns.md) -- side-by-side code translations for linear flows, joins, `foreach`, Parameters, IncludeFile, retries, compute, and runtime APIs
- [references/gaps-and-flags.md](references/gaps-and-flags.md) -- must-flag patterns, behavioral differences, decision tree, and migration refusal rules

### ZenML documentation

For questions beyond the migration surface itself, use the current ZenML documentation at https://docs.zenml.io.
