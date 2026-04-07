---
name: sagemaker-to-zenml-migration
description: >-
  Migrate Amazon SageMaker Pipelines and workflow code to idiomatic ZenML
  pipelines. Handles concept mapping (Pipeline->@pipeline,
  ProcessingStep/TrainingStep->@step, PropertyFile/JsonGet->artifacts),
  code translation, SagemakerOrchestratorSettings mapping, scheduling,
  model-registration strategy, and flags unsupported or high-risk patterns
  (CallbackStep, LambdaStep handshake semantics, step.properties placeholders,
  dynamic-pipeline scheduling on SageMaker) for human review. Use this skill
  whenever the user mentions SageMaker migration, converting SageMaker
  Pipelines, porting workflow code from SageMaker, replacing SageMaker DSL
  authoring with ZenML, or asks how a SageMaker Pipelines concept maps to
  ZenML -- even if they do not explicitly say "migrate". Also use when they
  paste `sagemaker.workflow.*` code and ask to make it work with ZenML, or
  when they describe a workflow using SageMaker terms (`ProcessingStep`,
  `TrainingStep`, `ConditionStep`, `PropertyFile`, `JsonGet`, `ModelStep`,
  `PipelineSession`) in a ZenML context. If the user just asks a quick
  conceptual question ("what's the ZenML equivalent of PropertyFile?"), answer
  it directly from the concept map -- no need to run the full migration
  workflow.
---

# Migrate SageMaker Pipelines to ZenML

This skill translates Amazon SageMaker Pipelines and related workflow code into idiomatic ZenML pipelines. It handles the full migration workflow: analyzing the SageMaker pipeline, classifying each construct, translating what maps cleanly, flagging what needs redesign, and producing a working ZenML project.

## How migration works at a high level

SageMaker Pipelines and ZenML can run on the same AWS substrate, but they ask you to think about pipelines differently. Native SageMaker Pipelines is a **step-type DSL**: you choose `ProcessingStep`, `TrainingStep`, `TuningStep`, `TransformStep`, `ConditionStep`, and other managed-service wrappers, then wire them with pipeline parameters, step `.properties`, `PropertyFile`, and `JsonGet`.

ZenML is **Python-step-first**: you write `@step` functions that consume and produce typed artifacts, and the active stack decides where they run. When you use ZenML's SageMaker orchestrator, you are usually **not leaving SageMaker**. You are keeping SageMaker as the execution backend while changing the authoring model to ZenML pipelines and stack settings.

That means migration is not a rename-the-primitives exercise. Some patterns translate directly, some are approximate, and some require deliberate redesign.

### The three mapping types

Every SageMaker concept falls into one of these categories:

| Type | Meaning | Action |
|------|---------|--------|
| **Direct** | Clean 1:1 mapping exists | Translate automatically |
| **Approximate** | Conceptual equivalent exists but semantics differ | Translate with caveats noted in migration report |
| **Absent** | No ZenML equivalent | Flag for human review with redesign suggestions |

See [references/concept-map.md](references/concept-map.md) for the full mapping tables.

## The Migration Workflow

### Phase 1: Receive and Analyze the SageMaker Workflow

Ask the user for their SageMaker pipeline code and any helper modules used by the pipeline. Useful inputs include:

- The main pipeline Python file (`Pipeline(...)`, `pipeline.upsert()`, `pipeline.start()`)
- Helper code used by processors, estimators, or model-registration logic
- Evaluation code that emits JSON reports or uses `PropertyFile` / `JsonGet`
- Scheduling snippets (`PipelineSchedule`, `put_triggers`, EventBridge role setup)
- Any deployment, registry, Clarify, QualityCheck, Feature Store, or callback code

Read everything thoroughly before doing anything else. For each pipeline, identify:

1. **Step inventory** -- Which step types are used? (`ProcessingStep`, `TrainingStep`, `TransformStep`, `TuningStep`, `ConditionStep`, `FailStep`, `ModelStep`, `CreateModelStep`, `RegisterModel`, `CallbackStep`, `LambdaStep`, `QualityCheckStep`, `ClarifyCheckStep`, `EMRStep`, `NotebookJobStep`, `AutoMLStep`)
2. **Data flow** -- Where are `Parameter*`, step `.properties`, `PropertyFile`, `JsonGet`, `ExecutionVariables`, or raw S3 URI handoffs used?
3. **Control flow** -- Are there `ConditionStep` branches, multiple condition types, or failure gates?
4. **Runtime and infrastructure** -- Are instance types, IAM roles, warm pools, S3 channels, tags, retries, or schedule roles embedded in pipeline code?
5. **AWS service coupling** -- Does the pipeline depend on SageMaker Model Registry, Experiments, Feature Store, Clarify, Model Monitor, Lambda, SQS callbacks, EMR, or Notebook Jobs?
6. **Migration target choice** -- Is the best path "portable ZenML first", or "keep SageMaker explicitly" using the SageMaker orchestrator and/or AWS SDK calls inside steps?

### Phase 2: Classify and Plan

For each component identified in Phase 1, classify it using the mapping type (direct / approximate / absent). Use the decision logic below and the full tables in [references/concept-map.md](references/concept-map.md).

#### Quick classification guide

**Direct translations (translate automatically):**
- `Pipeline` -> `@pipeline`
- `ParameterString` / `ParameterInteger` / `ParameterFloat` / `ParameterBoolean` -> typed pipeline parameters
- `ProcessingStep` -> `@step`
- `TrainingStep` -> `@step`
- `FailStep` -> raise an exception in a step

**Approximate translations (translate with caveats):**
- `ConditionStep` -> Python control flow, often `@pipeline(dynamic=True)` plus `.load()` for small control artifacts (note that dynamic pipelines are still experimental, and SageMaker-orchestrator scheduling does not support them)
- `PropertyFile` / `JsonGet` intent -> typed artifacts and normal Python access
- S3 input/output channels -> `SagemakerOrchestratorSettings`
- cache / retry behavior -> ZenML caching plus `StepRetryConfig`, while noting that SageMaker service-specific retry semantics may differ
- `TuningStep`, `TransformStep`, `QualityCheckStep`, `ClarifyCheckStep`, `EMRStep`, `NotebookJobStep`, `AutoMLStep` -> ZenML `@step` that explicitly launches the corresponding AWS-native behavior
- `ModelStep`, `CreateModelStep`, `RegisterModel` -> explicit SageMaker model / registry calls in steps, or an intentional redesign to ZenML Model Control Plane
- SageMaker Experiments -> experiment tracker component and/or explicit SageMaker experiment calls

**Absent / needs redesign (flag for human review):**
- Exact step `.properties` placeholder semantics
- Exact `PropertyFile` / `JsonGet` backend-evaluation behavior
- Native `CallbackStep` token-handshake semantics
- Native `LambdaStep` semantics when the original design relies on its I/O and timeout constraints
- Dynamic pipeline scheduling on the SageMaker orchestrator
- Any claim that SageMaker Model Registry == ZenML MCP
- Any claim that SageMaker Feature Store is usefully covered today by the ZenML feature-store abstraction

#### Present the migration plan

Before writing any code, present a summary to the user:

> "Here's what I found in your SageMaker pipeline:
> - **Direct translations** (will migrate cleanly): [list]
> - **Approximate translations** (will work but with noted caveats): [list]
> - **Needs redesign** (cannot auto-migrate safely): [list with brief explanation]
>
> Shall I proceed with the migration?"

If there are HIGH-severity flags, explain each one concretely: what the SageMaker code does, why ZenML cannot replicate it directly, and what the recommended redesign looks like.

### Phase 3: Generate ZenML Code

Translate the SageMaker workflow into a ZenML project. Follow these conventions strictly.

#### Project structure

Every migrated project MUST use this layout:

```
migrated_pipeline/
├── steps/                    # One file per step
│   ├── preprocess.py
│   ├── train.py
│   └── evaluate.py
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
- Always generate `configs/dev.yaml` AND `configs/prod.yaml`
- Always generate a `README.md` explaining the migrated pipeline and what still needs human review
- Include a brief ASCII DAG diagram in the pipeline file's module docstring
- Run `zenml init` at project root

#### Translation patterns

For each SageMaker construct, apply the appropriate translation. See [references/code-patterns.md](references/code-patterns.md) for detailed side-by-side examples covering the major migration patterns.

**The core translation rule**: move the pipeline's real business logic into `@step` functions with typed inputs and outputs. Use artifacts for data flow. Use `SagemakerOrchestratorSettings` only for runtime concerns you intentionally want to preserve on SageMaker.

```python
# SageMaker: step-type DSL + placeholder-driven wiring
# ZenML: explicit Python step interface + artifact wiring

@step
def evaluate(model_uri: str) -> dict[str, float]:
    return {"accuracy": 0.93}

@pipeline
def training_pipeline(model_uri: str) -> None:
    metrics = evaluate(model_uri)
    register_model(metrics=metrics)
```

**`PropertyFile` / `JsonGet` -> Artifact passing**: replace JSONPath-style placeholder extraction with structured return values:

```python
@step
def evaluate_model() -> dict[str, float]:
    return {"accuracy": 0.93, "f1": 0.91}

@pipeline(dynamic=True)
def gated_pipeline() -> None:
    metrics = evaluate_model()
    if metrics.load()["accuracy"] >= 0.9:
        deploy_model(metrics=metrics)
```

Use `.load()` only for **small control artifacts**. For large datasets or models, keep the value as an artifact edge and make decisions on a summarized artifact instead.

**Keep-SageMaker runtime mapping**: when preserving SageMaker runtime behavior, move instance types, S3 channels, warm pools, tags, and similar infrastructure knobs into `SagemakerOrchestratorSettings`, not business-logic parameters.

#### Dependency guidance

Do not force every migrated project to depend on the full SageMaker SDK.

- Always include the baseline ZenML requirements
- Add `boto3` and/or `sagemaker` only if the migrated project still makes AWS-native service calls or uses SageMaker-specific settings imports
- If the migration becomes fully portable ZenML, keep the dependency set small

#### Code comment style

Keep migration-related comments concise and actionable. Use `# Migration note:` for brief inline caveats and `# TODO(migration):` for items requiring user action. Put the long explanation in `MIGRATION_REPORT.md`, not in code comments.

#### Handling approximate translations

When translating approximate patterns, add a short comment explaining what changed:

```python
@step
def register_in_sagemaker(model_package_group: str, model_data_uri: str) -> str:
    # Migration note: the original pipeline used SageMaker ModelStep/RegisterModel.
    # This ZenML step keeps AWS-native registration explicitly instead of pretending
    # ZenML MCP is a 1:1 replacement for SageMaker Model Registry.
    ...
```

#### Handling absent patterns

For patterns that have no ZenML equivalent, do NOT silently approximate them. Instead:

1. Add a clearly marked `# TODO(migration)` comment in the generated code
2. Include the pattern in the migration report
3. Suggest a redesign approach

```python
# TODO(migration): UNSUPPORTED -- the original pipeline used CallbackStep token
# handshake semantics. ZenML has no equivalent native callback step. Keep this
# portion as an explicit AWS workflow (for example Step Functions or an AWS API
# step) or redesign the orchestration boundary.
@step
def request_external_review() -> None:
    ...
```

### Phase 4: Produce the Migration Report

After generating the ZenML project, produce a `MIGRATION_REPORT.md` in the project root:

```markdown
# Migration Report: [SageMaker Pipeline Name] -> [ZenML Pipeline Name]

## Summary
- **Source**: SageMaker pipeline `[pipeline_name]`
- **Target**: ZenML pipeline `[target_pipeline_name]`
- **Steps migrated**: X direct, Y approximate, Z flagged

## Direct Translations
| SageMaker Construct | ZenML Equivalent | Notes |
|---|---|---|
| ProcessingStep `Preprocess` | `steps/preprocess.py` | Clean translation to `@step` |

## Approximate Translations
| SageMaker Construct | ZenML Equivalent | What Changed |
|---|---|---|
| ConditionStep `Gate` | dynamic pipeline branch | Backend placeholder evaluation became Python control flow |
| RegisterModel | explicit registration step | Kept SageMaker registry call explicit |

## Flagged for Review
| SageMaker Pattern | Severity | Issue | Suggested Redesign |
|---|---|---|---|
| CallbackStep | HIGH | No native callback-token step in ZenML | Keep as explicit AWS workflow or redesign boundary |

## SageMaker Runtime Mapping
| Native SageMaker concern | ZenML equivalent | Notes |
|---|---|---|
| Instance type / volume | `SagemakerOrchestratorSettings` | Runtime concern, not business logic |
| S3 channels | `SagemakerOrchestratorSettings` | Preserve only when exact AWS semantics matter |
| Model Registry | explicit SageMaker step or ZenML MCP | Choose intentionally |

## Scheduling
- **Original**: [native schedule / trigger details]
- **Migrated**: [ZenML `Schedule(...)` or manual trigger design]
- **Note**: SageMaker orchestrator supports cron / interval / one-time schedules, but dynamic pipelines cannot be scheduled there

## Limitations and Key Differences
[Put the most important semantic differences here before the benefits section.]

## What's NOT Migrated
[List SageMaker-native service behavior that was intentionally left AWS-specific or requires manual review.]

## What You Get for Free After Migration
- **Artifact lineage and versioning**
- **Step caching**
- **Stack abstraction**
- **Experiment tracker integrations**
- **Model Control Plane**
- **Service connectors**

## Recommended Next Steps
1. Run the `zenml-quick-wins` skill
2. Install the ZenML docs MCP server
3. Review the flagged items with the linked docs
4. Use `zenml-pipeline-authoring` for deeper customization
```

### Phase 5: Suggest Next Steps

After migration is complete, always include a "Recommended Next Steps" section in the migration report AND communicate it to the user.

#### 1. Run the `zenml-quick-wins` skill

Always suggest this as the immediate next step:

> "Now that the migration is done, I'd recommend running the `zenml-quick-wins` skill to add metadata logging, experiment tracking, alerters, and other production features to your pipeline."

#### 2. Documentation links for flagged patterns

For every flagged pattern, include links to the relevant ZenML docs:

- SageMaker orchestrator: `https://docs.zenml.io/stacks/stack-components/orchestrators/sagemaker`
- Scheduling: `https://docs.zenml.io/concepts/steps_and_pipelines/scheduling`
- Dynamic pipelines: `https://docs.zenml.io/concepts/steps_and_pipelines/dynamic_pipelines`
- Service connectors / auth: `https://docs.zenml.io/how-to/infrastructure-deployment/auth-management/aws-service-connector`
- Artifact and stack concepts: `https://docs.zenml.io/stacks`
- Models / MCP: `https://docs.zenml.io/how-to/model-management-metrics/model-control-plane/register-a-model`

When the migrated project intentionally preserves AWS-native service calls, include AWS documentation links for those preserved service-specific patterns too.

#### 3. Suggest installing the ZenML docs MCP server

> "For easier access to ZenML documentation while you work, you can install the ZenML docs MCP server: `claude mcp add zenmldocs --transport http https://docs.zenml.io/~gitbook/mcp`"

#### 4. Community support for unsupported patterns

When the migration has HIGH-severity flags -- patterns that could not be directly migrated -- offer to help the user get support from the ZenML community. **When there are 2+ HIGH-severity flags**, generate a pre-made Slack message for `zenml.io/slack` summarizing the SageMaker pipeline, the unsupported patterns, the attempted workarounds, and the concrete question for the ZenML team.

#### 5. Open GitHub issues for genuine feature gaps

When the migration reveals a genuine missing feature in ZenML, offer to open a GitHub issue on `zenml-io/zenml` using `gh issue create`. Include the SageMaker pattern, the attempted workaround, and why the feature would help real migrations.

#### 6. Run `/simplify` to clean up the migrated code

After migration is complete, always suggest running `/simplify` on the generated code. Migration often leaves behind caveat comments, duplicated wrappers, and portability notes that should be tightened up before the code feels production-ready.

#### 7. Further customization via `zenml-pipeline-authoring`

The `zenml-pipeline-authoring` skill handles deeper customization:
- Docker settings for remote execution
- YAML configuration for multi-environment setups
- Custom materializers for domain-specific types
- Deployment and post-migration refinement

## Important Behavioral Differences to Communicate

These are the most common sources of confusion after migration. Always mention the relevant ones in the migration report.

### Step-type DSL != uniform ZenML steps

SageMaker pipeline code encodes AWS service choices directly in the graph: `ProcessingStep`, `TrainingStep`, `TransformStep`, `TuningStep`, and so on. ZenML uses a uniform `@step` abstraction, so service-specific behavior often moves **inside** the step body as an explicit SDK call or into orchestrator settings.

### Placeholder expressions != artifacts

SageMaker uses backend-evaluated placeholders such as step `.properties`, `PropertyFile`, and `JsonGet`. ZenML uses typed artifacts and normal Python values. This changes:
- Where values are evaluated
- How data is serialized
- What gets cached
- How downstream control flow is expressed

### Scheduling lifecycle differs

ZenML can schedule SageMaker-orchestrated pipelines, but schedule lifecycle management is not the same as native SageMaker. Dynamic pipelines are supported on the SageMaker orchestrator in general, but they cannot currently be scheduled there. Re-running a scheduled pipeline creates a new SageMaker pipeline / schedule rather than updating the old one.

### Model governance semantics differ

SageMaker Model Registry and SageMaker Experiments are not 1:1 with ZenML MCP and experiment trackers. For SageMaker Feature Store specifically, do **not** present the ZenML feature-store abstraction as a practical out-of-the-box replacement. If the user needs first-class SageMaker Feature Store support, keep the AWS-native usage explicit and suggest discussing the need in `zenml.io/slack` or contributing an integration.

## Anti-Patterns in Migration

| Anti-pattern | Why it's wrong | What to do instead |
|---|---|---|
| Keeping `PropertyFile` / `JsonGet` just to mimic the old DSL | Recreates placeholder complexity and loses most ZenML benefits | Return typed artifacts and index them normally |
| Mapping infrastructure parameters to ordinary business-logic args | Blurs runtime config with pipeline semantics | Move instance types, S3 channels, tags, warm pools into orchestrator settings |
| Pretending SageMaker Model Registry == ZenML MCP | They overlap in intent, not in semantics | Choose a registry strategy explicitly |
| Pretending SageMaker Feature Store is covered by the ZenML feature-store abstraction | That overstates current support and sets the user up for the wrong migration plan | Keep SageMaker Feature Store explicit, or discuss a custom/community integration path |
| Translating `ConditionStep` without noting control-flow changes | Backend placeholder evaluation and Python branching behave differently | Flag the change and use dynamic pipelines carefully |
| Silently collapsing `TuningStep` into one training step | Destroys original HPO behavior | Keep an explicit HPO step design |
| Claiming Lambda / callback primitives are preserved | They are not native ZenML concepts | Keep them as explicit AWS integrations or redesign |

## References

### Detailed reference files

- [references/concept-map.md](references/concept-map.md) -- Full concept mapping tables, scheduling caveats, and stack-component mappings
- [references/code-patterns.md](references/code-patterns.md) -- Side-by-side code translations for core SageMaker patterns
- [references/gaps-and-flags.md](references/gaps-and-flags.md) -- Must-flag patterns, behavioral differences, migration decision tree, and anti-patterns

### ZenML documentation

For topics beyond migration (stack setup, deployment, experiment tracking, model management), query the ZenML docs at https://docs.zenml.io.
