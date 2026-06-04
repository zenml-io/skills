---
name: argo-to-zenml-migration
description: >-
  Migrate Argo Workflows, WorkflowTemplates, ClusterWorkflowTemplates, and
  CronWorkflows to idiomatic ZenML pipelines. Handles concept mapping,
  YAML-to-Python translation, scheduling, retries, Kubernetes-native pattern
  analysis, and flags unsupported patterns such as status-based depends logic,
  shared volumes, containerSet, sidecars, synchronization locks, and Argo
  Events for human review. Use this skill whenever the user mentions Argo
  migration, converting Argo YAML, replacing Argo with ZenML, mapping an Argo
  concept to ZenML, or provides workflow YAML using terms like
  WorkflowTemplate, CronWorkflow, when, withItems, withParam, containerSet,
  onExit, Sensor, or EventSource. For quick conceptual questions, answer from
  the concept map without running the full migration workflow.
---

# Migrate Argo Workflows to ZenML

This skill translates Argo Workflows into idiomatic ZenML pipelines. It handles the full migration workflow: analyzing Argo YAML and referenced scripts, classifying each pattern, translating what maps cleanly, flagging what needs redesign, and producing a working ZenML project.

## How migration works at a high level

Argo and ZenML can both express DAG-shaped workflows, but they speak different native languages.

- **Argo** is a Kubernetes-native workflow engine driven by YAML CRDs, template references, variable substitution, pod topology, and controller policies.
- **ZenML** is a Python-first pipeline framework driven by `@step`, `@pipeline`, typed artifacts, stack components, and orchestrator-managed execution.

That means this migration is not a simple rename operation. Some Argo features map well, some only preserve intent, and some depend so heavily on controller or pod semantics that they must be redesigned explicitly.

### The three mapping types

Every Argo concept falls into one of these categories:

| Type | Meaning | Action |
|------|---------|--------|
| **Direct** | Clean 1:1 or near-1:1 mapping exists | Translate automatically |
| **Approximate** | Conceptual equivalent exists but semantics differ | Translate with caveats noted in the migration report |
| **Absent** | No ZenML equivalent exists | Flag for human review with redesign suggestions |

See [references/concept-map.md](references/concept-map.md) for the full mapping tables.

## The Migration Workflow

### Phase 1: Receive and Analyze the Argo Workflow

Ask the user for all relevant migration inputs. At minimum, request:

- the Argo YAML manifests (`Workflow`, `WorkflowTemplate`, `ClusterWorkflowTemplate`, `CronWorkflow`)
- any referenced scripts, Python modules, container commands, or shell scripts
- any EventSource / Sensor / trigger definitions if Argo Events is involved
- any Kubernetes resources the workflow depends on (PVCs, service accounts, secrets, ConfigMaps, pod-spec patches)
- any assumptions about cluster-local files, mounted volumes, or helper containers

Read everything thoroughly before doing anything else. For each workflow, identify:

1. **Object kinds** -- `Workflow`, `WorkflowTemplate`, `ClusterWorkflowTemplate`, `CronWorkflow`, `EventSource`, `Sensor`
2. **Template types** -- `container`, `script`, `dag`, `steps`, `resource`, `http`, `suspend`, `plugin`, `data`, `containerSet`
3. **Dependencies** -- `dependencies`, enhanced `depends`, `templateRef`, `workflowTemplateRef`, workflow-of-workflows patterns
4. **Data flow** -- workflow parameters, input parameters, output parameters from files, `outputs.result`, artifacts, `globalName`
5. **Control flow** -- `when`, `withItems`, `withParam`, `withSequence`, `continueOn`, `onExit`, lifecycle hooks
6. **Execution policies** -- retries, backoff, deadlines, memoization, synchronization, fail-fast settings
7. **Kubernetes coupling** -- shared volumes, PVC templates, `emptyDir`, sidecars, init containers, node selectors, affinity, tolerations, service accounts
8. **Scheduling and triggers** -- `CronWorkflow` schedules, concurrency policy, timezone, Argo Events integration
9. **Runtime contract** -- whether a template is really value-based, path-based, or same-pod / same-filesystem coupled

### Phase 2: Classify and Plan

For each component identified in Phase 1, classify it using the mapping type (direct / approximate / absent). Use the decision logic below and the full tables in [references/concept-map.md](references/concept-map.md).

#### Quick classification guide

**Direct or near-direct translations (translate automatically):**
- workflow parameters -> pipeline parameters
- template input parameters -> step function arguments
- simple `dag` success-path dependencies -> pipeline step wiring
- `withSequence` -> Python `range(...)` in a dynamic pipeline
- retry count/delay -> `StepRetryConfig(...)`
- memoization intent -> ZenML caching

**Approximate translations (translate with caveats):**
- `Workflow` / `WorkflowTemplate` -> `@pipeline` plus a pipeline run or reusable Python module
- `CronWorkflow` -> OSS/orchestrator-backed `Schedule(...)` on a supported orchestrator, with `zenml pipeline schedule ...` for supported lifecycle operations; ZenML Pro schedule triggers are separate snapshot trigger objects
- `container` / `script` template -> `@step` with Python logic, `DockerSettings`, or `subprocess.run(...)`
- output parameter via file or `outputs.result` -> explicit step return value
- artifact passing -> ZenML artifacts and materializers
- `when` on values -> dynamic pipeline branching or explicit soft-conditional logic
- `withItems` / `withParam` -> dynamic fanout using real list artifacts and `.map()`; for dynamic pipelines, default to `STOP_ON_FAILURE`, use `FAIL_FAST` only with caveats, and do not recommend `CONTINUE_ON_FAILURE`
- `onExit` -> redesign using hooks, idempotent cleanup, execution modes, or external cleanup control
- `resource` / `http` templates -> explicit SDK or API calls inside steps
- Kubernetes resources / placement settings -> orchestrator-specific settings documented as portability loss

**Absent / redesign-first patterns (flag for human review):**
- enhanced `depends` expressions based on task status (`A.Failed`, `B.Errored`, etc.)
- `containerSet`
- `suspend` templates unless the target project is on a recent 0.94.x release with `zenml.wait(...)` support (preferably `>=0.94.1`), and even then only as an explicit redesign
- shared PVC / `emptyDir` filesystem contracts across steps
- sidecars, daemon containers, and same-pod helper services
- synchronization mutexes / semaphores
- Argo Events graphs (`EventSource` + `Sensor` + dependency logic); ZenML Pro platform-event triggers may cover supported ZenML platform lifecycle events, but they are not Argo Events graph parity
- non-Python arbitrary images that cannot be sensibly wrapped in a Python-capable ZenML step environment

#### Present the migration plan

Before writing any code, present a summary to the user:

> "Here's what I found in your Argo workflow:
> - **Direct translations** (will migrate cleanly): [list]
> - **Approximate translations** (will work but with noted caveats): [list]
> - **Needs redesign** (cannot auto-migrate): [list with brief explanation]
>
> Shall I proceed with the migration?"

If there are HIGH-severity flags, explain each one concretely: what the Argo workflow does, why ZenML cannot reproduce it directly, and what the redesign options look like.

### Phase 3: Generate ZenML Code

Translate the Argo workflow into a ZenML project. Follow these conventions strictly.

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

This matches the `zenml-pipeline-authoring` skill's conventions. Key rules:

- one step per file in `steps/`
- separate pipeline definition from execution
- `run.py` uses `argparse` (click conflicts with ZenML)
- `pyproject.toml` uses `requires-python = ">=3.12"` and `zenml>=0.94.1` as the dependency floor
- always generate `configs/dev.yaml` and `configs/prod.yaml`
- always generate a `README.md` explaining what was migrated and what still needs manual attention
- include concise migration comments in code, not essay-length explanations
- run `zenml init` at the project root

#### Translation patterns

See [references/code-patterns.md](references/code-patterns.md) for side-by-side examples covering DAGs, steps templates, loops, conditionals, artifacts, CronWorkflows, exit handlers, shared-volume redesigns, and Argo Events integration patterns.

**The core translation rule**: move business logic out of YAML templates and into typed Python functions. Treat the YAML graph as a description of orchestration, then rebuild that orchestration in Python using step calls and artifact wiring.

```python
# Argo mental model
# template "extract" writes a value to a file or stdout

# ZenML translation
@step
def extract() -> list[int]:
    return [1, 2, 3]
```

**Parameters -> function arguments**:

```python
@pipeline
def training_pipeline(dataset: str, learning_rate: float = 0.1) -> None:
    prepared = prepare_data(dataset=dataset)
    train_model(data=prepared, learning_rate=learning_rate)
```

**Output files / `outputs.result` -> step returns**:

```python
@step
def read_threshold() -> float:
    # Migration note: Argo previously read this from `valueFrom.path`.
    # ZenML treats the extracted value as a normal typed return.
    return 0.8
```

**File artifacts -> typed artifacts or explicit `Path` contracts**:

```python
from pathlib import Path

@step
def produce_manifest() -> Path:
    output = Path("/tmp/manifest.json")
    output.write_text("{\"ok\": true}")
    return output
```

Use a file- or path-based contract only when the original workflow genuinely depends on file identity or layout. Otherwise prefer typed domain objects.

**CronWorkflow -> Schedule**:

```python
from zenml.config.schedule import Schedule

schedule = Schedule(cron_expression="0 2 * * *")
my_pipeline.with_options(schedule=schedule)()
```

Always note that scheduling is orchestrator-dependent. In OSS, this is a `Schedule(...)` on a pipeline run, managed with singular `zenml pipeline schedule ...` commands where the orchestrator supports lifecycle operations. In ZenML Pro, schedule triggers are separate server-side trigger objects attached to snapshots (`zenml trigger schedule create`, `attach`, `list`, `delete`). CronWorkflow-specific behavior like timezone and concurrency policy may need extra handling.

**Argo Events -> trigger redesign**: Do not claim Argo Events parity. ZenML Pro platform-event triggers attach supported ZenML platform events to snapshots, but Argo `EventSource`/`Sensor` dependency graphs usually need external eventing plus a ZenML snapshot/API/deployment trigger design.

**Suspend / pause-resume note**:

If the source workflow uses `suspend`, do not present ZenML wait/resume as a generic drop-in replacement. Treat it as a redesign option that requires a recent 0.94.x release -- preferably `zenml>=0.94.1`, where `zenml.wait(...)` and pipeline run resume are documented in the official changelog.

#### Code comment style

Keep migration-related comments short and actionable:

- use `# Migration note:` for brief inline caveats
- use `# TODO(migration):` for unsupported patterns or manual follow-up
- keep long explanations in the migration report, not in code

#### Handling approximate translations

When translating approximate patterns, add a brief note explaining the semantic shift:

```python
@step
def run_cli(command: list[str]) -> str:
    # Migration note: Argo container templates can run any image + command.
    # This ZenML step runs in a Python-capable environment and wraps the CLI
    # with subprocess. Verify tooling is present in the step image.
    import subprocess

    completed = subprocess.run(command, check=True, text=True, capture_output=True)
    return completed.stdout
```

#### Handling absent patterns

For patterns that have no ZenML equivalent, do NOT silently approximate them. Instead:

1. add a clearly marked `# TODO(migration)` comment in the generated code
2. include the pattern in the migration report
3. suggest a redesign approach

```python
# TODO(migration): UNSUPPORTED -- Argo enhanced depends expression
# `cleanup` previously ran when `train.Failed || validate.Errored`.
# ZenML cannot branch on upstream failure states this way. Redesign with
# explicit status artifacts, hooks, or separate alerting/cleanup flows.
@step
def cleanup_after_failure(status: dict[str, bool]) -> None:
    ...
```

### Phase 4: Produce the Migration Report

After generating the ZenML project, produce a `MIGRATION_REPORT.md` in the project root.

```markdown
# Migration Report: [Argo Workflow] -> [ZenML Pipeline]

## Summary
- **Source**: Argo `[kind]` `[name]`
- **Target**: ZenML pipeline `[pipeline_name]`
- **Templates migrated**: X direct, Y approximate, Z flagged

## Direct Translations
| Argo Template / Concept | ZenML Equivalent | Notes |
|---|---|---|
| parameters | pipeline args | Clean translation |

## Approximate Translations
| Argo Template / Concept | ZenML Equivalent | What Changed |
|---|---|---|
| CronWorkflow | `Schedule(...)` or ZenML Pro schedule trigger | OSS schedule support depends on orchestrator and uses `zenml pipeline schedule ...`; Pro schedule triggers attach to snapshots; concurrency/timezone policy may need extra handling |

## Flagged for Review
| Argo Pattern | Severity | Issue | Suggested Redesign |
|---|---|---|---|
| containerSet | HIGH | No multi-container same-pod primitive in ZenML | Collapse into one step or externalize to Kubernetes-native service |

## Scheduling
- **Original**: CronWorkflow `0 2 * * *`, timezone `UTC`
- **Migrated OSS path**: `Schedule(cron_expression="0 2 * * *")`, managed with `zenml pipeline schedule ...` where supported
- **ZenML Pro option**: a schedule trigger attached to the target snapshot (`zenml trigger schedule create` + `zenml trigger schedule attach`)
- **Note**: Verify orchestrator scheduling support and any concurrency / timezone semantics

## Kubernetes-Native / Infrastructure Assumptions
| Original Argo Assumption | Migration Status | Notes |
|---|---|---|
| Shared PVC | Flagged | Redesign around artifacts or explicit external storage |

## Limitations and Key Differences
[Summarize the most important semantic shifts before listing benefits.]

## What's NOT Migrated
[List Argo features outside the portable ZenML scope: Sensors, EventSources, cluster policy, sidecars, etc.]

## What You Get for Free After Migration
- typed artifacts and artifact lineage
- step caching
- stack abstraction
- service connectors and secrets management
- Model Control Plane for ML workflows

## Recommended Next Steps
1. Run the `zenml-quick-wins` skill
2. Install the ZenML docs MCP server
3. Review every HIGH-severity redesign item
4. Use the `zenml-pipeline-authoring` skill for Docker settings, custom materializers, deployment, or deeper configuration
```

### Phase 5: Suggest Next Steps

After migration is complete, always include a "Recommended Next Steps" section in the migration report AND communicate it to the user.

#### 1. Run the `zenml-quick-wins` skill

Always suggest this as the immediate next step:

> "Now that the migration is done, I'd recommend running the `zenml-quick-wins` skill to add metadata logging, experiment tracking, alerters, and other production-readiness features."

#### 2. Documentation links for flagged patterns

For every flagged pattern, include a link to the relevant ZenML documentation:

- Scheduling: `https://docs.zenml.io/how-to/steps-pipelines/scheduling`
- Dynamic pipelines: `https://docs.zenml.io/how-to/steps-pipelines/dynamic-pipelines`
- ZenML Pro triggers: `https://docs.zenml.io/getting-started/zenml-pro/triggers`
- Orchestrators: `https://docs.zenml.io/stacks/stack-components/orchestrators`
- Containerization: `https://docs.zenml.io/how-to/containerization/containerization`
- Secrets management: `https://docs.zenml.io/how-to/secrets/secrets`
- Service connectors / auth: `https://docs.zenml.io/how-to/infrastructure-deployment/auth-management`

#### 3. Suggest installing the ZenML docs MCP server

> "For easier access to ZenML documentation while you work, you can install the ZenML docs MCP server: `claude mcp add zenmldocs --transport http https://docs.zenml.io/~gitbook/mcp`"

#### 4. Community support for unsupported patterns

When the migration has HIGH-severity flags, offer to help the user get support from the ZenML community.

**When there are 2+ HIGH-severity flags**, generate a pre-made Slack message for `zenml.io/slack` that includes:

- what they are migrating
- the unsupported Argo patterns
- a short code snippet for each relevant pattern
- what redesign ideas were already considered
- a clear ask for better approaches or feature guidance

```markdown
**Argo -> ZenML Migration Help**

I'm migrating an Argo Workflow that uses [patterns]. The migration skill flagged these as needing redesign:

1. **[Pattern]**: [brief description + Argo snippet]
   - Suggested workaround: [X]

2. **[Pattern]**: [brief description + Argo snippet]
   - Suggested workaround: [Y]

I'm looking for advice on whether there is a better ZenML-native pattern or an upcoming feature that would fit this use case.
```

#### 5. Open GitHub issues for genuine feature gaps

When the migration reveals a real missing feature in ZenML, offer to open a GitHub issue on `zenml-io/zenml` using `gh issue create`. Include the Argo pattern, why it matters, what workaround was attempted, and why the gap is broadly useful.

#### 6. Run `/simplify` to clean up the migrated code

Always suggest running `/simplify` after the migration:

> "The migration is done. I'd recommend running `/simplify` on the generated code to clean up migration comments, reduce duplication, and make the result feel more like native ZenML code."

#### 7. Further customization via `zenml-pipeline-authoring`

The `zenml-pipeline-authoring` skill handles deeper customization:

- Docker settings for remote execution
- YAML configuration for multi-environment setups
- custom materializers for file or domain-specific contracts
- pipeline deployments and serving

## Important Behavioral Differences to Communicate

These are the most common sources of confusion after migration. Always mention the relevant ones in the migration report.

### Declarative YAML != imperative Python

Argo YAML is the execution program. ZenML Python is the execution program. That means a migration rewrites orchestration into code; it does not merely reformat manifests.

### File-based artifacts != typed artifacts

In Argo, file paths, output files, and artifact repository wiring are often part of the workflow contract. In ZenML, the typed value is the contract, and storage is delegated to the artifact store plus materializers.

### Kubernetes-native != stack-abstracted

Argo assumes Kubernetes everywhere. ZenML can run on Kubernetes, but it treats Kubernetes as one execution backend among several. Pod-level settings need to be isolated as infrastructure concerns, not mixed into portable pipeline logic.

### Shared volumes / same-pod semantics != isolated steps

Argo patterns built around shared PVCs, `emptyDir`, sidecars, init containers, or same-pod container coordination do not map cleanly to independent ZenML steps. These are redesign hotspots.

### Status-based control flow != value-based control flow

Argo can branch on task result states directly. ZenML control flow is much more naturally expressed as value-based logic in Python. If the old behavior depends on failure states, model that explicitly and document the semantic change.

## Anti-Patterns in Migration

| Anti-pattern | Why it's wrong | What to do instead |
|---|---|---|
| Translating YAML templates 1:1 into step files without redesigning data flow | Preserves Argo syntax shape but not ZenML semantics | Design step interfaces around typed args, returns, and artifacts |
| Treating output parameters as ZenML pipeline parameters | Upstream-produced values are artifacts in ZenML | Return values from steps and wire them downstream |
| Returning `/tmp/...` paths from one step and assuming another step can read them | ZenML steps are isolated; local paths are not stable contracts | Return typed data, or explicitly model a file / URI contract |
| Reusing arbitrary non-Python images unchanged | ZenML step execution still expects a Python-capable runtime | Build a Python-capable image or keep that execution outside ZenML |
| Translating `depends: "A.Failed"` into a naive Python `if` after `A()` | A failing ZenML step changes run behavior; it is not just a boolean value | Model failure as data, use hooks, or split flows |
| Replacing shared-volume logic with `/tmp` across multiple steps | `/tmp` is container-local, not workflow-global | Collapse into one step or externalize state |
| Rewriting `withParam` as JSON strings passed between steps | Keeps Argo's serialization hack and loses ZenML typing | Return real `list[...]` artifacts and fan out dynamically |
| Mapping `onExit` to a success hook | Success hooks do not behave like workflow finalizers | Use `try/finally`, failure hooks, or external cleanup orchestration |
| Claiming Argo Events maps directly to native ZenML event graphs | ZenML Pro platform-event triggers are snapshot trigger objects for supported ZenML platform events, not Argo Events graph parity | Use supported ZenML Pro triggers where they fit; otherwise use external event infrastructure to invoke ZenML runs, snapshots, or deployment endpoints |

## References

### Detailed reference files

- [references/concept-map.md](references/concept-map.md) -- full concept mapping tables for Argo objects, template types, data passing, control flow, execution features, Kubernetes-native features, and Argo Events
- [references/code-patterns.md](references/code-patterns.md) -- side-by-side translation examples for the most common Argo migration patterns
- [references/gaps-and-flags.md](references/gaps-and-flags.md) -- must-flag patterns, behavioral differences, template classification, and the migration decision tree

### ZenML documentation

For topics beyond migration (stack setup, experiment tracking, deployment, or orchestrator setup), query the ZenML docs at https://docs.zenml.io.
