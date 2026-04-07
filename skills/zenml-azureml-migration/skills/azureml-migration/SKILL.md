---
name: azureml-to-zenml-migration
description: >-
  Migrate Azure Machine Learning SDK v2 pipelines, components, environments,
  and schedules to idiomatic ZenML pipelines. Handles concept mapping
  (`@pipeline` -> `@pipeline`, `@command_component` -> `@step`,
  `Environment(...)` -> `DockerSettings(...)`,
  AzureML compute -> `AzureMLOrchestratorSettings`), code translation,
  Azure-aware "keep AzureML" migration paths, and flags unsupported or unsafe
  patterns (sweep jobs, parallel jobs, managed endpoints, AzureML Registry,
  Responsible AI dashboard, and unverified control-flow helpers like
  `if_else` and `do_while`) for human review. Use this skill whenever the user
  mentions AzureML migration, Azure Machine Learning SDK v2 migration,
  converting AzureML pipelines or components, porting workflows from AzureML,
  replacing AzureML authoring with ZenML, or asks how AzureML concepts map to
  ZenML -- even if they don't explicitly say "migrate". Also use when they
  paste AzureML SDK v2 code, `mldesigner` components, YAML components,
  `load_component()` usage, MLTable/data asset definitions, or AzureML
  scheduling/deployment code and ask to make it work with ZenML. If the user
  just asks a quick conceptual question ("what's the ZenML equivalent of an
  AzureML Environment?"), answer it directly from the concept map -- no need
  to run the full migration workflow.
---

# Migrate AzureML SDK v2 Pipelines to ZenML

This skill translates **Azure Machine Learning SDK v2** pipelines into idiomatic ZenML pipelines. It handles the full migration workflow: analyzing AzureML pipeline code and assets, classifying each pattern, choosing when to keep AzureML as the execution plane, translating what maps cleanly, flagging what needs redesign, and producing a working ZenML project.

> SDK v1 is out of scope. If the user is still on AzureML SDK v1, first normalize the workflow to SDK v2 concepts, then migrate the SDK v2 version to ZenML.

## How migration works at a high level

AzureML and ZenML solve overlapping problems, but they organize the world differently:

- **AzureML authoring is asset-centric**: components, environments, data assets, model assets, and compute are all separate first-class objects.
- **ZenML authoring is Python-workflow-centric**: the main surface is `@step` + `@pipeline`, with containerization handled by `DockerSettings` and infrastructure handled by the active stack and orchestrator settings.

That means migration is not just a syntax rewrite. Some AzureML ideas translate directly, some translate approximately, and some should stay Azure-native or be redesigned.

### The "keep AzureML" migration path

This is not always a story of leaving Azure. Often the best migration is:

1. Keep **AzureML as the execution plane**
2. Move **workflow authoring** into ZenML
3. Preserve Azure compute, AzureML Studio job inspection, workspace security context, and Azure scheduling where it still makes sense

In that setup, ZenML authors the workflow, and the AzureML orchestrator submits AzureML jobs under the hood.

### If you choose the "keep AzureML" path, confirm prerequisites first

Before promising a keep-Azure migration, verify that the target ZenML stack can actually run on the AzureML orchestrator. The official ZenML AzureML orchestrator docs call out these prerequisites:

- the ZenML `azure` integration installed
- Docker running locally **or** a remote image builder in the stack
- a **remote artifact store**
- a **remote container registry**
- an Azure resource group with an **AzureML workspace**
- Azure authentication configured for the orchestrator

For authentication, default authentication can be fine during development, but **service principal authentication is the recommended production path**. If the user wants to keep AzureML execution, ask early whether these stack prerequisites already exist. If they do not, say so plainly in the migration plan instead of making it sound like the code translation alone is sufficient.

### The three mapping types

Every AzureML concept falls into one of these categories:

| Type | Meaning | Action |
|------|---------|--------|
| **Direct** | Clean 1:1 mapping exists | Translate automatically |
| **Approximate** | Conceptual equivalent exists but semantics differ | Translate with caveats noted in the migration report |
| **Absent** | No ZenML equivalent | Flag for human review with redesign suggestions |

See [references/concept-map.md](references/concept-map.md) for the full mapping tables.

## The Migration Workflow

### Phase 1: Receive and Analyze the AzureML Workflow

Ask the user for the AzureML assets that actually define the workflow. They may provide:

- Python SDK v2 pipeline code (`@pipeline`, `@command_component`, `PipelineJob`)
- `mldesigner` components
- YAML component definitions used with `load_component()`
- environment definitions (`Environment(...)`, curated environment names, Docker build contexts)
- compute configuration (serverless, compute instances, compute clusters, node-level overrides)
- data/model asset references (`uri_file`, `uri_folder`, `mltable`, model assets)
- schedule definitions (`JobSchedule`, cron, recurrence)
- deployment code (managed online endpoints, batch endpoints)

Read everything thoroughly before doing anything else. For each workflow, identify:

1. **Pipeline graph** -- Which nodes exist, and how are outputs wired into downstream inputs?
2. **Component types** -- Are components Python-defined, YAML-defined, registered, or pipeline components?
3. **Environment usage** -- Does each component own its own environment asset? Are there custom images, Conda files, or Docker build contexts?
4. **Compute bindings** -- Is compute set at the pipeline level, node level, or both? Are there heterogeneous per-node requirements?
5. **Data and model asset semantics** -- Are asset IDs and versions part of the business logic, or are they just storage details?
6. **MLTable usage** -- Is MLTable just a convenient input type, or does the workflow depend on MLTable-specific semantics?
7. **Advanced job types** -- Any sweep jobs, parallel jobs, Spark jobs, AutoML jobs, import/data transfer jobs?
8. **Scheduling** -- Cron or recurrence? Is the user expecting ZenML to fully manage schedule lifecycle after creation?
9. **Deployment flows** -- Does the workflow register models or deploy managed online/batch endpoints?
10. **Platform dependencies** -- Registry sharing, Responsible AI dashboard, Designer, workspace-specific networking, or other Azure-only features?
11. **Target stack prerequisites** -- If the plan is to keep AzureML execution, does the user already have the Azure integration, remote artifact store, remote container registry, AzureML workspace, and authentication/service connector setup needed by the AzureML orchestrator?
12. **Control flow helpers** -- Any use of `if_else`, `do_while`, `parallel_for`, or `set_pipeline_controller_configurations`?

### Phase 2: Classify and Plan

For each component identified in Phase 1, classify it as direct / approximate / absent using [references/concept-map.md](references/concept-map.md).

#### Quick classification guide

**Direct translations (translate automatically):**
- `@pipeline` -> `@pipeline`
- `@command_component` -> `@step`
- Pipeline parameters/defaults -> typed Python parameters/defaults
- Primitive AzureML inputs -> normal Python parameters
- AzureML serverless / compute instance / compute cluster -> `AzureMLOrchestratorSettings(mode=...)`
- Azure cron / recurrence schedules -> ZenML `Schedule(...)` **when the AzureML orchestrator is the target**
- MLflow tracking inside AzureML -> MLflow usage inside ZenML steps

**Approximate translations (translate with caveats):**
- `Environment(image=..., conda_file=...)` -> `DockerSettings(...)`
- YAML components loaded via `load_component()` -> rewrite as Python `@step`
- `Input` / `Output` asset types -> step parameters and return values
- `node.compute = ...` -> step-specific runtime config, step operator settings, or separate pipelines
- MLTable/data assets -> URI boundaries, artifacts, or explicit Azure asset IDs passed through as strings
- Pipeline components/sub-pipelines -> helper pipeline functions
- Model registration -> ZenML model object plus optional Azure registration step
- AutoML -> call Azure AutoML from a ZenML step, or replace with explicit training/HPO

**Absent / needs redesign (flag for human review):**
- Sweep jobs as a native ZenML primitive
- Parallel jobs as a native ZenML primitive
- Managed online endpoints / batch endpoints as native ZenML deployment primitives
- AzureML Registry and registered component registry semantics
- Responsible AI dashboard
- Designer
- Cross-workspace asset sharing semantics
- `if_else`, `do_while`, `parallel_for`, `set_pipeline_controller_configurations` as safe auto-translations

#### Present the migration plan

Before writing any code, present a summary to the user:

> "Here's what I found in your AzureML workflow:
> - **Direct translations** (will migrate cleanly): [list]
> - **Approximate translations** (will work but with caveats): [list]
> - **Needs redesign / stays Azure-native**: [list with brief explanation]
>
> The key architectural choice is whether to keep AzureML as the execution plane via the AzureML orchestrator, or to move execution to another ZenML stack. Shall I proceed with the migration?"

If there are HIGH-severity flags, explain each one concretely:

- what the AzureML code does today
- why ZenML cannot reproduce it literally
- whether the right answer is **keep Azure-native**, **call the Azure SDK from a step**, or **redesign the workflow**

### Phase 3: Generate ZenML Code

Translate the AzureML workflow into a ZenML project. Follow these conventions strictly.

#### Project structure

Every migrated project MUST use this layout:

```text
migrated_pipeline/
├── steps/                    # One file per step
│   ├── extract.py
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

Key rules:

- One step per file in `steps/`
- Separate pipeline definition from execution
- `run.py` uses `argparse`
- `pyproject.toml` uses `requires-python = ">=3.12"`
- Always generate both `configs/dev.yaml` and `configs/prod.yaml`
- Always generate a `README.md` explaining what stayed Azure-native and what requires manual attention
- If helpful, include a small ASCII DAG diagram in the pipeline module docstring
- Run `zenml init` at the project root

#### Translation patterns

See [references/code-patterns.md](references/code-patterns.md) for detailed side-by-side examples.

The core translation rule is:

1. Move AzureML component logic into a typed `@step`
2. Move environment definitions into `DockerSettings`
3. Move compute choices into orchestrator settings (or explicit runtime config)
4. Replace asset edge wiring with step return values and step inputs

```python
# AzureML
@command_component(name="train_model")
def train_component(input_data: Input(type="uri_folder"), epochs: int = 10):
    ...

# ZenML
@step
def train_step(input_data_uri: str, epochs: int = 10) -> str:
    ...
```

#### Environment translation rule

AzureML environments are not first-class ZenML assets. Translate them into Docker/container settings:

```python
from zenml.config import DockerSettings

docker_settings = DockerSettings(
    parent_image="mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu20.04",
    requirements=["mlflow", "scikit-learn"],
)
```

If the AzureML environment used a custom Docker build context, preserve that with `dockerfile=...` and `build_context_root=...`.

#### Compute translation rule

If the user wants to **keep AzureML execution**, prefer `AzureMLOrchestratorSettings`:

```python
from zenml.integrations.azure.flavors import AzureMLOrchestratorSettings

cluster = AzureMLOrchestratorSettings(
    mode="compute-cluster",
    compute_name="gpu-cluster",
    size="Standard_NC6s_v3",
    min_instances=0,
    max_instances=4,
)
```

Use this as the default migration path whenever preserving Azure compute behavior is important.

#### Data and asset translation rule

AzureML asset-typed inputs/outputs (`uri_file`, `uri_folder`, `mltable`, model assets) do **not** map to first-class ZenML asset descriptors. Decide case by case:

- If the asset identity matters, keep the Azure asset ID/URI explicit
- If only the data matters, translate to artifacts or loaded Python objects
- If the workflow crosses Azure-native boundaries, keep the boundary explicit in the migrated code

#### Scheduling rule

ZenML can create AzureML cron/interval schedules when using the AzureML orchestrator, but ZenML does **not** fully manage schedule lifecycle on Azure afterward. Always call this out in the migration report.

#### Handling approximate translations

When translating approximate patterns, add concise inline comments describing the changed behavior:

```python
@step(settings={"docker": docker_settings})
def train_step(input_data_uri: str) -> str:
    # Migration note: the original AzureML component used an Environment asset.
    # This step uses DockerSettings instead, so environment reuse/versioning now
    # lives in code and image configuration rather than in Azure asset registry state.
    ...
```

#### Handling absent patterns

For patterns with no safe ZenML equivalent:

1. Add a `# TODO(migration):` comment in the generated code
2. Include the item in the migration report
3. State clearly whether the user should:
   - keep the Azure feature external,
   - call the Azure SDK from a ZenML step, or
   - redesign the workflow

```python
# TODO(migration): UNSUPPORTED -- AzureML parallel job semantics are not a safe
# 1:1 ZenML translation. Keep the Azure parallel job native, or redesign this
# workload as a dynamic ZenML pipeline only after validating data partitioning,
# concurrency, and retry semantics.
```

### Phase 4: Produce the Migration Report

After generating the ZenML project, produce a `MIGRATION_REPORT.md` in the project root:

```markdown
# Migration Report: [AzureML Workflow] -> [ZenML Pipeline]

## Summary
- **Source**: AzureML SDK v2 workflow `[name]`
- **Target**: ZenML pipeline `[pipeline_name]`
- **Components migrated**: X direct, Y approximate, Z flagged
- **Execution target**: AzureML orchestrator / other ZenML stack

## Direct Translations
| AzureML Concept | ZenML Equivalent | Notes |
|---|---|---|

## Approximate Translations
| AzureML Concept | ZenML Equivalent | What Changed |
|---|---|---|

## Flagged for Review
| AzureML Pattern | Severity | Issue | Recommended Path |
|---|---|---|---|

## Environment Translation Summary
| AzureML Environment | ZenML DockerSettings | Notes |
|---|---|---|

## Compute Mapping
| AzureML Compute | ZenML Equivalent | Notes |
|---|---|---|

## Scheduling
- Original schedule
- Migrated schedule
- Lifecycle caveat for AzureML-managed schedules

## Limitations and Key Differences

## What's NOT Migrated

## What You Get for Free After Migration

## Recommended Next Steps
```

### Phase 5: Suggest Next Steps

After migration is complete, always include a "Recommended Next Steps" section in the report and communicate it to the user.

#### 1. Run `zenml-quick-wins`

Always suggest this first:

> "Now that the migration is done, I'd recommend running the `zenml-quick-wins` skill to add metadata logging, experiment tracking, and other production features to your pipeline."

#### 2. Link the user to the right ZenML docs

For flagged patterns, include targeted docs:

- Scheduling: `https://docs.zenml.io/how-to/steps-pipelines/schedule-a-pipeline`
- Orchestrators: `https://docs.zenml.io/stacks/stack-components/orchestrators`
- AzureML orchestrator: `https://docs.zenml.io/stacks/stack-components/orchestrators/azureml`
- Dynamic pipelines: `https://docs.zenml.io/how-to/steps-pipelines/dynamic-pipelines`
- Containerization: `https://docs.zenml.io/how-to/containerization/containerization`
- Secrets management: `https://docs.zenml.io/how-to/project-setup-and-management/secret-management`
- Auth / service connectors: `https://docs.zenml.io/how-to/infrastructure-deployment/auth-management`
- Models / MCP: `https://docs.zenml.io/how-to/model-management-and-deployment`

#### 3. Suggest the ZenML docs MCP server

> "For easier access to ZenML documentation while you work, you can install the ZenML docs MCP server: `claude mcp add zenmldocs --transport http https://docs.zenml.io/~gitbook/mcp`"

#### 4. Offer community support for genuine gaps

When there are multiple HIGH-severity flags, offer to draft a Slack message for `zenml.io/slack` summarizing:

- what AzureML workflow is being migrated
- which patterns had no safe ZenML equivalent
- what workaround or redesign was attempted
- what answer the user needs from the ZenML team

#### 5. Offer GitHub issue creation for real product gaps

If the migration reveals a genuine feature gap in ZenML, offer to open an issue on `zenml-io/zenml`.

#### 6. Suggest `/simplify`

After code generation, suggest running `/simplify` to reduce migration-only comments and clean up redundant patterns.

#### 7. Use `zenml-pipeline-authoring` for deeper customization

Recommend `zenml-pipeline-authoring` when the user needs:

- advanced Docker settings
- YAML-driven environments
- custom materializers
- deployment workflows
- more complex stack-specific runtime configuration

## Important Behavioral Differences to Communicate

These are the places where users are most likely to get surprised after migration.

### Asset model vs code model

AzureML treats components, environments, and many data/model artifacts as first-class registered assets. ZenML treats the main reusable unit as Python code plus build/runtime configuration. This changes how reuse, versioning, and governance are expressed.

### Environment management

AzureML `Environment` objects become `DockerSettings`. The operational goal is similar, but the management surface is different:

- less Azure-native registry semantics
- more code-local, container-oriented configuration

### Compute management

AzureML pipeline DSL allows very explicit node-level compute choices. ZenML can preserve many Azure compute settings with the AzureML orchestrator, but strongly heterogeneous per-node compute often needs a design decision instead of a literal rewrite.

### Data handling

AzureML uses typed asset descriptors like `uri_folder` and `mltable`. ZenML uses Python parameters, artifacts, and materializers. Similar intent, different model.

### Schedule lifecycle

ZenML can create AzureML schedules, but it does not fully manage their lifecycle after creation on AzureML. Updating or deleting those schedules may remain a manual Azure task.

### Control flow

Treat `if_else`, `do_while`, `parallel_for`, and `set_pipeline_controller_configurations` as unsafe for automatic translation unless the user explicitly wants a redesign and accepts manual review.

## Anti-Patterns in Migration

| Anti-pattern | Why it's wrong | What to do instead |
|---|---|---|
| Converting AzureML assets into ZenML names without changing the architecture | Keeps Azure mental model but loses correctness | Rewrite around `@step`, `@pipeline`, artifacts, and orchestrator settings |
| Treating every AzureML Environment as a reusable ZenML asset | ZenML has no environment asset registry primitive | Move environment intent into `DockerSettings` and document the change |
| Silently converting sweep jobs into Python loops | Changes execution semantics, observability, and retry behavior | Keep Azure sweep native, or redesign with explicit HPO tooling |
| Silently converting parallel jobs into `.map()` | Similar shape, different guarantees | Flag for review and redesign deliberately |
| Pretending MLTable is a first-class ZenML type | It is not | Keep it as a URI/reference boundary or load it explicitly inside a step |
| Dropping managed endpoint logic during migration | Changes production behavior, not just authoring | Keep endpoint deployment in Azure or call Azure SDK from a step |
| Claiming `if_else` / `do_while` are safe 1:1 conversions | They are not verified here | Flag them and require manual review |
| Assuming ZenML will fully manage Azure schedules after creation | It will not | Document the lifecycle caveat in the migration report |

## References

### Detailed reference files

- [references/concept-map.md](references/concept-map.md) -- Full mapping tables for pipeline concepts, component types, environments, compute, data assets, scheduling, model management, and Azure platform features
- [references/code-patterns.md](references/code-patterns.md) -- Side-by-side AzureML -> ZenML code patterns for components, environments, compute, scheduling, assets, deployment, and redesign-only patterns
- [references/gaps-and-flags.md](references/gaps-and-flags.md) -- Must-flag patterns, behavioral differences, migration decision tree, and anti-patterns

### ZenML documentation

For topics beyond migration (stack setup, experiment tracking, deployment, authentication), query the ZenML docs at https://docs.zenml.io.
