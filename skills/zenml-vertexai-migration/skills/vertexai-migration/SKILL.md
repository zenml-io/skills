---
name: vertexai-to-zenml-migration
description: >-
  Migrate Vertex AI Pipelines (Kubeflow Pipelines v2 / PipelineJob workflows)
  to idiomatic ZenML pipelines. Handles concept mapping
  (`@dsl.pipeline` -> `@pipeline`, `@dsl.component` -> `@step`,
  `PipelineJob.create_schedule(...)` -> `Schedule(...)`), artifact-contract
  translation (`Input[Dataset]`, `InputPath`, `.uri`, `.path`),
  Google Cloud Pipeline Components (GCPC) rewrites, dynamic control flow
  (`dsl.If`, `dsl.ParallelFor`, `dsl.Collected`), resource/config migration,
  and flags unsupported patterns (compiled template workflows, `dsl.ExitHandler`,
  path-coupled artifacts, schedule lifecycle parity) for human review. Use this
  skill whenever the user mentions Vertex AI Pipelines migration, KFP v2 to
  ZenML, PipelineJob migration, GCPC migration, or asks how a Vertex/KFP concept
  maps to ZenML — even if they do not explicitly say "migrate". Also use when
  they paste KFP DSL code, compiled pipeline YAML/JSON, Vertex submission code,
  or describe a workflow using Vertex/KFP terminology (`dsl.component`,
  `dsl.pipeline`, `dsl.If`, `dsl.ParallelFor`, `PipelineJob`, GCPC) in a ZenML
  context. If the user just asks a quick conceptual question ("what is the
  ZenML equivalent of `dsl.importer`?"), answer it directly from the concept map
  — no need to run the full migration workflow.
---

# Migrate Vertex AI Pipelines to ZenML

This skill translates **Vertex AI Pipelines / KFP v2 / PipelineJob workflows** into idiomatic ZenML pipelines.

The shape looks familiar at first: both systems use decorated Python functions, typed inputs and outputs, and DAG-style orchestration. But under the hood the story is different:

- **KFP / Vertex** compiles Python into a pipeline template and treats artifacts as runtime-managed references with URIs and local paths.
- **ZenML** executes from Python definitions and treats artifacts as Python values that are materialized into an artifact store.

That means this migration is not a search-and-replace job. Some patterns map directly, some map approximately, and some must be treated as redesign boundaries.

## How migration works at a high level

For most teams, the safest migration story is:

1. **Keep Vertex as the execution plane**
2. **Add ZenML as the control plane**
3. **Rewrite KFP / Vertex authoring concepts into ZenML-native pipelines**

There are two practical strategies:

- **Preferred**: full ZenML-native rewrite running on the ZenML Vertex orchestrator
- **Fallback**: keep the compiled KFP template and wrap the `PipelineJob` submission inside a ZenML step as a temporary black-box migration

The skill should optimize for the first strategy and document the second honestly as a partial migration escape hatch.

### The three mapping types

| Type | Meaning | Action |
|------|---------|--------|
| **Direct** | Clean 1:1 mapping exists | Translate automatically |
| **Approximate** | Similar intent, different semantics | Translate with caveats and record the difference |
| **Absent** | No ZenML equivalent | Flag for human review and suggest a redesign |

See [references/concept-map.md](references/concept-map.md) for the full mapping tables.

## The Migration Workflow

### Phase 1: Receive and analyze the Vertex / KFP workflow

Ask the user for the actual source artifacts before rewriting anything. In practice that may be:

- KFP DSL source files
- Vertex submission code using `PipelineJob(...)`
- compiled YAML / JSON pipeline templates
- GCPC usage
- component code and helper modules

Read all of it carefully and identify:

1. **Pipeline structure** — `@dsl.pipeline`, subpipelines, graph helpers, imported components
2. **Component types** — plain Python `@dsl.component`, `@dsl.container_component`, importer components, GCPC operators
3. **Artifact contract** — `Input[T]`, `Output[T]`, `InputPath`, `OutputPath`, `.path`, `.uri`, metadata helpers
4. **Control flow** — `dsl.If`, `dsl.Elif`, `dsl.Else`, `dsl.Condition`, `dsl.ParallelFor`, `dsl.Collected`, `dsl.ExitHandler`
5. **Compile / template workflow** — `compiler.Compiler().compile()`, template registries, `template_path=...`, schedule creation from compiled templates
6. **Scheduling** — `PipelineJob.create_schedule(...)`, cron strings, timezones, start/end windows, concurrency knobs
7. **Infrastructure / runtime settings** — base image, packages, target image, CPU / memory / GPU settings, service accounts, network, persistent resources, labels, caching
8. **Platform integrations** — Vertex Experiments, Model Registry, Endpoints, BigQuery, Dataflow, Dataproc, GCS, TensorBoard

### Phase 2: Classify and plan

Classify each concept as direct / approximate / absent using [references/concept-map.md](references/concept-map.md) and [references/gaps-and-flags.md](references/gaps-and-flags.md).

#### Quick classification guide

**Direct translations (translate automatically):**
- `@dsl.pipeline` -> `@pipeline`
- typed pipeline parameters -> typed pipeline function args
- plain Python `@dsl.component` -> `@step` when the contract is value-centric
- simple dataflow wiring -> step outputs passed directly into downstream step inputs

**Approximate translations (translate with caveats):**
- KFP artifacts -> typed ZenML artifacts / materializers
- `InputPath` / `OutputPath` -> explicit file handling or a location-aware artifact contract
- `dsl.If` on upstream outputs -> `@pipeline(dynamic=True)` + `.load()`
- `dsl.ParallelFor` -> `.map()` in a dynamic pipeline
- `dsl.Collected` -> reducer step over mapped outputs
- importer components -> `ExternalArtifact(...)` or explicit artifact lookup
- `PipelineJob.create_schedule(...)` -> `Schedule(...)`
- resource setters -> `ResourceSettings(...)` plus Vertex-specific orchestrator settings
- Vertex Experiments / Model Registry / Endpoint ops -> explicit SDK-calling steps

**Absent / must-flag patterns (never silently approximate):**
- GCPC operator catalog parity
- compiled-template publishing workflows
- `dsl.ExitHandler`
- schedule lifecycle parity from ZenML back into Vertex
- automatic Vertex Model Registry parity
- automatic Vertex Experiment parity
- artifact contracts that fundamentally depend on KFP-managed paths / URIs / placeholders

#### Present the migration plan

Before writing code, summarize the findings for the user:

> "Here is what I found in your Vertex / KFP pipeline:
> - **Direct translations**: [list]
> - **Approximate translations**: [list]
> - **Needs redesign**: [list]
>
> The main risk areas are [artifact contract / GCPC / control flow / template lifecycle]. Shall I proceed with the migration?"

If there are high-risk flags, explain them concretely in story form: what the original KFP code does, why ZenML cannot preserve that behavior 1:1, and what the least-bad redesign looks like.

### Phase 3: Generate ZenML code

Translate the workflow into a ZenML project. Follow these conventions strictly.

#### Project structure

Every migrated project MUST use this layout:

```text
migrated_pipeline/
├── steps/                    # One file per step
│   ├── extract.py
│   ├── train.py
│   └── deploy.py
├── pipelines/
│   └── my_pipeline.py
├── materializers/            # Custom materializers when path/URI semantics matter
├── configs/
│   ├── dev.yaml
│   └── prod.yaml
├── run.py                    # CLI entry point (argparse, not click)
├── README.md
└── pyproject.toml
```

Key rules:

- One step per file in `steps/`
- Keep pipeline definition separate from execution
- `run.py` uses `argparse`
- `pyproject.toml` uses `zenml>=0.94.1` and `requires-python = ">=3.12"`
- Always generate `configs/dev.yaml` and `configs/prod.yaml`
- Always generate a `README.md`
- Run `zenml init` at the project root

#### Translation rules that matter most

**1. Artifact-contract-first translation**

Do not start by translating decorators. Start by asking what the step *really* exchanges.

- If `.path` / `.uri` is only a serialization boundary, convert to a normal Python type and let ZenML materialize it.
- If the code depends on location identity, directory layout, sibling files, or passing artifact paths to an external binary, keep that as an explicit contract and use a custom materializer, reference object, or controlled file-handling pattern.

**2. GCPC rewrite rule**

Never pretend ZenML has a matching GCPC operator.

- GCPC nodes must become ordinary ZenML steps
- Those steps should call the relevant SDK / API directly
- Mark every such translation as a **rewrite**, not a direct translation

**3. Control-flow rule**

- Static `if` is only safe when the condition depends on pipeline parameters
- Runtime branching on upstream outputs requires `@pipeline(dynamic=True)` plus `.load()`
- `dsl.ParallelFor` should map to `.map()` only when the resulting dynamic-pipeline semantics are acceptable

**4. Compile / template rule**

- Do not keep `compiler.Compiler().compile()` in migrated ZenML code
- If the user must keep template-based submission, that is a **partial migration**
- In that partial migration, wrap the existing `PipelineJob` submission in a ZenML step and be explicit that ZenML sees it as one black-box node

**5. Comment style**

Use concise migration comments:

- `# Migration note:` for semantic caveats
- `# TODO(migration):` for required user action

Keep detailed prose in the migration report, not in the code.

#### Handling approximate translations

When semantics change, leave a short inline note:

```python
@step
def evaluate_model(metrics_path_hint: str | None = None) -> dict[str, float]:
    # Migration note: the original KFP component wrote metrics to a managed
    # artifact path. This ZenML step returns metrics as a typed artifact instead.
    return {"accuracy": 0.91}
```

#### Handling absent patterns

For unsupported patterns:

1. Add a `# TODO(migration):` comment in code
2. Record it in the migration report
3. Suggest a redesign

```python
# TODO(migration): UNSUPPORTED — original pipeline relied on dsl.ExitHandler
# to guarantee cleanup after failure. ZenML has no exact equivalent.
# Redesign this as idempotent cleanup plus hooks / external failure handling.
```

### Phase 4: Produce the migration report

After generating the project, always create `MIGRATION_REPORT.md` in the project root.

Use this structure:

```markdown
# Migration Report: [Vertex Pipeline] -> [ZenML Pipeline]

## Summary
- **Source**: Vertex AI Pipelines / KFP v2 workflow `[name]`
- **Target**: ZenML pipeline `[pipeline_name]`
- **Steps migrated**: X direct, Y approximate, Z flagged

## Direct Translations
| Source Concept | ZenML Target | Notes |

## Approximate Translations
| Source Concept | ZenML Target | What Changed |

## Flagged for Review
| Pattern | Severity | Issue | Suggested Redesign |

## Artifact Contract Decisions
| Source Component | Old Contract | New Contract | Notes |

## GCPC Rewrite Summary
| GCPC Node | Replacement Step | SDK / API Used | Notes |

## Vertex Platform Integration Mapping
| Vertex Feature | ZenML Mapping | Notes |

## Compiled Template / Schedule Lifecycle Gaps
- [list exactly what is not preserved]

## What's NOT Migrated
- template registries
- Vertex schedule lifecycle management
- Vertex Model Registry parity unless explicitly implemented
- Vertex Experiment wiring unless explicitly configured
- ML Metadata equivalence

## What You Get for Free After Migration
- artifact versioning and lineage
- stack portability
- step caching
- model / artifact control-plane capabilities
- a cleaner separation between pipeline logic and cloud SDK calls

## Recommended Next Steps
1. Run `zenml-quick-wins`
2. Install the ZenML docs MCP server
3. Validate the migrated pipeline on the ZenML Vertex orchestrator
4. Run the same pipeline on a second stack if portability matters
5. Run `/simplify` on the generated code
```

### Phase 5: Suggest next steps

After migration, always guide the user toward the next layer of cleanup and hardening.

#### 1. Run `zenml-quick-wins`

Say this explicitly:

> "Now that the migration is done, I recommend running `zenml-quick-wins` to add metadata logging, experiment tracking, secrets, and other production features."

#### 2. Link the user to the right ZenML docs

When relevant, include specific links:

- Vertex orchestrator: `https://docs.zenml.io/stacks/stack-components/orchestrators/vertex`
- Scheduling: `https://docs.zenml.io/how-to/steps-pipelines/scheduling`
- Dynamic pipelines: `https://docs.zenml.io/how-to/steps-pipelines/dynamic-pipelines`
- External artifacts: `https://docs.zenml.io/user-guides/starter-guide/manage-artifacts#consuming-external-artifacts-within-a-pipeline`
- Materializers: `https://docs.zenml.io/concepts/artifacts/materializers`
- Experiment trackers: `https://docs.zenml.io/stacks/stack-components/experiment-trackers`

#### 3. Suggest the docs MCP server

> "For easier access to current ZenML docs while you refine the migration, install the docs MCP server: `claude mcp add zenmldocs --transport http https://docs.zenml.io/~gitbook/mcp`"

#### 4. Offer community escalation for real gaps

When there are **2+ HIGH-severity flags**, generate a copy-pasteable Slack message for `zenml.io/slack` summarizing:

- what is being migrated
- which patterns failed to map cleanly
- what redesigns were attempted
- what help the user needs

#### 5. Offer GitHub issue creation for genuine product gaps

If the migration exposes a real missing feature in ZenML, offer to open an issue on `zenml-io/zenml` with:

- the original Vertex / KFP pattern
- why it matters
- what workaround was attempted
- what a useful feature would look like

#### 6. Suggest `/simplify`

Migration output is often correct but bulky. Always suggest `/simplify` to remove scaffolding comments, reduce duplication, and make the code feel production-ready.

## Important behavioral differences to communicate

These are the places where users are most likely to think "this looks the same" when it is not the same.

### Artifact model

KFP artifacts are runtime-managed references with `.path`, `.uri`, and metadata attached. ZenML artifacts are Python values loaded and saved by materializers.

Practical consequence: a KFP `Input[Dataset]` is not automatically the same as a ZenML `pd.DataFrame`. Sometimes it is. Sometimes that translation erases the real contract.

### Pipeline execution model

KFP authoring usually ends in compilation and submission of a template. ZenML runs from Python definitions through its orchestrator abstraction.

Practical consequence: compiled-template workflows, template registries, and `template_path=...` usage are redesign boundaries.

### Control flow

KFP runtime branching and fan-out are backend orchestration concepts. ZenML can do similar work with dynamic pipelines, but the mechanics differ.

Practical consequence: never translate `dsl.If` into a plain Python `if` in a static pipeline, and never translate `dsl.ParallelFor` into a plain `for` loop if you need backend fan-out semantics.

### Scheduling

Vertex exposes richer schedule lifecycle and concurrency controls than ZenML’s generic `Schedule(...)` surface.

Practical consequence: cron schedule creation may migrate, but full lifecycle parity usually does not.

### Experiments, model registry, and metadata

Vertex Experiments, Vertex Model Registry, and Vertex ML Metadata overlap with ZenML concepts, but they are not the same objects or the same metadata plane.

Practical consequence: migrate the *intent* explicitly. Do not tell the user that ZenML artifacts automatically become Vertex Model resources or that ZenML metadata is the same thing as Vertex MLMD.

## Anti-patterns in migration

| Anti-pattern | Why it is wrong | What to do instead |
|---|---|---|
| Translating every `Input[Dataset]` to `pd.DataFrame` | Erases URI/path semantics when the original component was location-aware | Classify the artifact contract first |
| Pretending GCPC has ZenML equivalents | ZenML has no GCPC-style operator catalog | Rewrite as SDK-calling steps |
| Keeping `compiler.Compiler().compile()` in migrated code | ZenML does not need user-facing compilation | Remove it, or treat compiled-template submission as a partial migration |
| Replacing `dsl.ParallelFor` with a plain `for` loop | Loses backend fan-out and observability | Use dynamic pipelines with `.map()` when appropriate |
| Replacing runtime `dsl.If` with static Python branching | Changes control-flow semantics | Use `dynamic=True` plus `.load()` |
| Treating `dsl.ExitHandler` as "just add a last step" | Exit handlers can run after failure; a last step may never run | Redesign cleanup / notification semantics explicitly |
| Assuming ZenML model artifacts equal Vertex Model resources | They are different resource models | Add an explicit model-upload step if Vertex Model Registry matters |
| Assuming ZenML schedule updates delete/update Vertex schedules | ZenML does not fully manage Vertex schedule lifecycle | Document manual schedule management |
| Assuming caching semantics are identical | Cache identity and execution model differ | Revalidate caching behavior after migration |

## References

### Detailed reference files

- [references/concept-map.md](references/concept-map.md) — Full concept mapping tables, scheduling support notes, and platform feature mapping
- [references/code-patterns.md](references/code-patterns.md) — Concrete KFP / Vertex -> ZenML translation patterns for the major migration surfaces
- [references/gaps-and-flags.md](references/gaps-and-flags.md) — Must-flag patterns, artifact contract classification, behavioral differences, and migration decision tree

### ZenML documentation

For topics beyond migration, query the ZenML docs at https://docs.zenml.io.
