---
name: dagster-to-zenml-migration
description: >-
  Migrate Dagster assets, ops, graphs, jobs, and software-defined asset
  workflows to idiomatic ZenML pipelines. Handles concept mapping
  (asset->step output, job->pipeline, IOManager->artifact store/materializer +
  explicit IO steps), asset-boundary planning, code translation, scheduling,
  retry config, resources/config migration, and flags unsupported patterns
  (asset selection, partitions/backfills, sensors, declarative automation,
  freshness policies, observable source assets) for human review. Use this
  skill whenever the user mentions Dagster migration, converting Dagster assets
  or jobs, porting workflows from Dagster, replacing Dagster with ZenML, or
  asks how a Dagster concept maps to ZenML -- even if they do not explicitly
  say "migrate". Also use when they paste Dagster code and ask to make it work
  with ZenML, or when they describe a workflow using Dagster terminology
  (`@asset`, `@multi_asset`, `Definitions`, `IOManager`, `ConfigurableResource`,
  partitions, sensors, asset checks) in a ZenML context. If the user just asks
  a quick conceptual question ("what is the ZenML equivalent of an IOManager?"
  or "how should I think about Dagster assets in ZenML?"), answer it directly
  from the concept map -- no need to run the full migration workflow.
---

# Migrate Dagster to ZenML

This skill translates Dagster projects into idiomatic ZenML pipelines. It handles the full migration workflow: analyzing Dagster code, classifying each pattern, deciding where Dagster asset boundaries become ZenML pipeline boundaries, translating what maps cleanly, flagging what needs redesign, and producing a working ZenML project.

## How migration works at a high level

Dagster and ZenML are both orchestration systems, but they organize work around different primary objects.

- **Dagster** increasingly centers the world around **assets**: named data products, asset checks, partitions, asset selections, and policies that decide when assets should materialize.
- **ZenML** centers the world around **pipelines and steps**: Python-defined execution graphs that produce typed artifacts, backed by stack components and artifact lineage.

That means a Dagster -> ZenML migration is not mainly a decorator rename. The hard part is semantic:

- deciding what the true **execution unit** is,
- deciding where a Dagster asset graph should become **one pipeline vs multiple pipelines**, and
- deciding which orchestration features are safe to preserve vs which must be redesigned.

Think of it like moving a library into a workshop. In Dagster, the shelves themselves are the first-class object. In ZenML, the assembly line is the first-class object. The books may stay the same, but the floor plan changes.

### The three mapping types

Every Dagster concept falls into one of these categories:

| Type | Meaning | Action |
|------|---------|--------|
| **Direct** | Clean 1:1 mapping exists | Translate automatically |
| **Approximate** | Conceptual equivalent exists but semantics differ | Translate with caveats noted in migration report |
| **Absent** | No ZenML equivalent | Flag for human review with redesign suggestions |

See [references/concept-map.md](references/concept-map.md) for the full mapping tables.

## The Migration Workflow

### Phase 1: Receive and Analyze the Dagster Code

Ask the user for their Dagster codebase, or the relevant files if it is too large. Read the code thoroughly before doing anything else. Inventory the project in two passes:

#### 1. Classify the project style

Determine whether the codebase is primarily:

- **Asset-centric** -- mostly `@asset`, `@multi_asset`, `define_asset_job`, asset checks, partitions, and asset automation
- **Op/job-centric** -- mostly `@op`, `@graph`, `@job`
- **Mixed** -- asset graph plus legacy ops/jobs or helper graphs

This matters because `@op` / `@job` code usually maps more cleanly to ZenML than asset-heavy code.

#### 2. Inventory the important Dagster patterns

For each module, identify:

1. **Core primitives** -- `@asset`, `@multi_asset`, `@graph_asset`, `@graph_multi_asset`, `@op`, `@graph`, `@job`, `Definitions`
2. **Dependency structure** -- asset deps, graph composition, asset jobs, asset selection
3. **Data and IO** -- `IOManager`, `ConfigurableIOManager`, `SourceAsset`, observable source assets, metadata, asset checks
4. **Config and resources** -- `Config`, `RunConfig`, `ConfigurableResource`, `EnvVar`
5. **Execution semantics** -- partitions, partition mappings, backfills, schedules, sensors, asset sensors, declarative automation, freshness policies
6. **Dynamic behavior** -- `DynamicOut`, `DynamicOutput`, runtime fan-out, asset subsets, dynamic partitions
7. **Infrastructure** -- executors, launchers, Docker/Kubernetes settings, resource tags
8. **Testing patterns** -- how Dagster execution and checks are currently tested

When the codebase uses asset-heavy features, open [references/gaps-and-flags.md](references/gaps-and-flags.md) early. That file is the safety rail for migration.

### Phase 2: Classify and Plan

For each component identified in Phase 1, classify it using the mapping type (direct / approximate / absent). Use the quick guide below and the full tables in [references/concept-map.md](references/concept-map.md).

#### Quick classification guide

**Direct translations (translate automatically):**
- `@op` -> `@step`
- simple `@job` -> `@pipeline`
- `RetryPolicy` -> `StepRetryConfig`
- typed config values -> typed pipeline/step parameters

**Approximate translations (translate with caveats):**
- `@asset` -> step output artifact inside a pipeline
- `@multi_asset` -> multi-output step
- `@graph_asset` -> helper steps plus a terminal output artifact
- `ConfigurableResource` -> stack components + secrets + service connectors + step-local helper objects
- schedules -> OSS/orchestrator-backed `Schedule(...)` on supported orchestrators, with `zenml pipeline schedule ...` for supported lifecycle operations; ZenML Pro schedule triggers are separate server-side trigger objects attached to snapshots
- `IOManager` -> artifact store/materializer plus explicit source/sink steps
- `SourceAsset` -> `ExternalArtifact` or explicit source-loading step
- asset-check logic -> validation step body, but without Dagster's independently managed check-node semantics
- `DynamicOutput` fan-out -> dynamic pipeline or explicit redesign; dynamic pipelines default to `STOP_ON_FAILURE`, support `FAIL_FAST` with caveats, and do not support `CONTINUE_ON_FAILURE`

**Absent / needs redesign (flag for human review):**
- asset selection and subset materialization semantics
- partition mappings and non-trivial partition/backfill behavior
- declarative automation / auto-materialize policies
- freshness policies as first-class orchestration rules
- sensors and asset sensors; ZenML Pro platform-event triggers may cover supported ZenML platform lifecycle events, but Dagster sensor cursors and arbitrary sensor evaluation logic need redesign
- observable source assets as first-class graph nodes
- IO managers that embed business logic beyond serialization
- `@multi_asset` subset semantics

#### Mandatory pipeline-boundary decision

Before writing any code, make an explicit decision about the migration shape. This is the single most important Dagster-specific step.

Choose one of these:

1. **Single ZenML pipeline**
   - Use only when the Dagster code already behaves like one tightly coupled execution unit.
   - Common for op/job-centric projects or very small asset graphs.

2. **Multiple ZenML pipelines in one migrated project**
   - Use when the original Dagster project relies on asset selection, different schedules, different ownership boundaries, or distinct backfill domains.
   - This is often the honest choice for asset-heavy code.

3. **Partial migration + flagged redesign**
   - Use when unsupported Dagster semantics dominate.
   - In this case, generate only the safe core, add `# TODO(migration)` markers, and make the redesign requirements explicit.

Present this decision clearly to the user before generating code:

> "Here is the migration shape I recommend:
> - **Pipeline boundary decision**: [single pipeline / multiple pipelines / partial migration]
> - **Why**: [concrete explanation tied to the Dagster code]
> - **Direct translations**: [list]
> - **Approximate translations**: [list]
> - **Needs redesign**: [list with brief explanation]
>
> Shall I proceed with this migration plan?"

If there are HIGH-severity flags, explain each one concretely: what the Dagster code does, why ZenML cannot replicate it directly, and what redesign would preserve the intent most honestly.

### Phase 3: Generate ZenML Code

Translate the Dagster project into a ZenML project. Follow these conventions strictly.

#### Project structure

Every migrated project MUST use this layout:

```text
migrated_pipeline/
├── steps/                    # One file per step
├── pipelines/
│   ├── __init__.py
│   ├── main_pipeline.py
│   └── extra_pipeline.py     # If the Dagster project becomes multiple pipelines
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
- `run.py` uses `argparse`
- `pyproject.toml` uses `zenml>=0.94.1` and `requires-python = ">=3.12"`
- Run `zenml init` at the project root
- Always generate `configs/dev.yaml` and `configs/prod.yaml`
- Always generate a `README.md` explaining the migrated pipeline(s), how to run them, and what still needs manual attention
- Add concise `# Migration note:` comments for semantic differences
- Add `# TODO(migration):` comments only where genuine redesign work remains

#### Multiple pipelines are allowed

Unlike Airflow- or Databricks-style migrations, a Dagster migration may honestly need **multiple** ZenML pipelines. Do not force a single pipeline just for symmetry.

#### `run.py` behavior

- If exactly one pipeline is generated, `run.py` may run that pipeline by default.
- If multiple pipelines are generated, `run.py` should expose a `--pipeline` argument so the user can choose which pipeline entry point to run.
- If partitions or operational slices mattered in Dagster, `run.py` should also expose the relevant parameters (`--partition-key`, `--start-date`, `--end-date`, etc.).

#### Core translation rule

Move the compute body into a `@step` function, type-hint the inputs and outputs, and wire steps through function calls in a `@pipeline`.

See [references/code-patterns.md](references/code-patterns.md) for side-by-side examples covering:
- asset graphs
- op/job workflows
- IO managers
- resources/config
- partitions
- schedules
- sensors
- asset checks
- `@multi_asset`, `@graph_asset`, and dynamic fan-out

#### Handling approximate translations

When translating approximate patterns, add brief inline comments in the generated code explaining the semantic difference:

```python
@step
def load_orders() -> pd.DataFrame:
    # Migration note: the original Dagster asset could be materialized
    # independently inside the asset graph. In ZenML this data is produced
    # as part of a pipeline run and persisted as an artifact.
    ...
```

#### Handling absent patterns

For patterns that have no ZenML equivalent, do NOT silently approximate them. Instead:

1. Add a clearly marked `# TODO(migration)` comment in the generated code
2. Include the pattern in the migration report
3. Suggest a redesign approach

Example:

```python
# TODO(migration): UNSUPPORTED -- Dagster asset selection / subset materialization
# was part of the original workflow here. ZenML does not support first-class
# asset selection semantics. Recommended redesign: split the asset group into
# separate pipelines and load shared upstream results via ExternalArtifact or
# an explicit source step.
```

### Phase 4: Produce the Migration Report

After generating the ZenML project, produce a `MIGRATION_REPORT.md` in the project root:

```markdown
# Migration Report: [Dagster Project] -> [ZenML Project]

## Summary
- **Source**: Dagster project `[name]`
- **Target**: ZenML project `[name]`
- **Project style**: asset-centric / op-job-centric / mixed
- **Components migrated**: X direct, Y approximate, Z flagged

## Pipeline Boundary Decisions
| Dagster run unit / asset slice | ZenML pipeline | Why split or combine |
|---|---|---|
| daily_orders assets | pipelines/orders_daily.py | Dagster users materialized this slice independently |

## Direct Translations
| Dagster Component | ZenML Component | Notes |
|---|---|---|
| `train_model` op | `steps/train_model.py` | Clean op -> step translation |

## Approximate Translations
| Dagster Component | ZenML Component | What Changed |
|---|---|---|
| `cleaned_orders` asset | `steps/clean_orders.py` | Asset became a step output artifact inside a pipeline |
| warehouse IOManager | `steps/load_orders.py` + artifact store | Business logic moved from IO manager into explicit step |

## Flagged for Review
| Dagster Pattern | Severity | Issue | Suggested Redesign |
|---|---|---|---|
| Asset selection | HIGH | No first-class subset materialization in ZenML | Split into multiple pipelines |
| Daily partitions + partition mappings | HIGH | No native partition engine | Explicit partition-key params + external backfill driver |
| Sensor cursor | HIGH | No Dagster-style sensor/cursor API | External event service, ZenML Pro platform-event trigger for supported ZenML platform events, or snapshot/API/deployment trigger redesign |

## IO / Storage Migration
[Summarize what was preserved and what moved out of IO managers]

## Partition / Backfill Strategy
[Explain how partition keys and backfills are handled after migration]

## Automation and Scheduling Gaps
[Explain schedules, sensors, freshness, declarative automation, and what changed. Distinguish OSS/orchestrator schedules (`Schedule(...)` plus `zenml pipeline schedule ...` where supported) from ZenML Pro schedule/platform-event trigger objects attached to snapshots. Do not present Dagster sensors, sensor cursors, declarative automation, or freshness policies as 1:1 ZenML equivalents.]

## What's NOT Migrated
[List the Dagster semantics or platform features left outside the migrated code]

## What You Get for Free After Migration
- **Artifact versioning and lineage**
- **Step caching**
- **Stack abstraction**
- **Service connectors**
- **Model Control Plane** (if relevant)

## Recommended Next Steps
1. Run the `zenml-quick-wins` skill for metadata logging, experiment tracking, and alerters
2. Install the ZenML docs MCP server: `claude mcp add zenmldocs --transport http https://docs.zenml.io/~gitbook/mcp`
3. Follow the docs links for flagged patterns
4. Use `zenml-pipeline-authoring` for deeper customization
```

### Phase 5: Suggest Next Steps

After migration is complete, always include a "Recommended Next Steps" section in the migration report AND communicate it to the user.

#### 1. Run the `zenml-quick-wins` skill

Always suggest this as the immediate next step:

> "Now that the migration is done, I recommend running the `zenml-quick-wins` skill to add metadata logging, experiment tracking, alerts, and other production improvements."

#### 2. Documentation links for flagged patterns

For every flagged pattern, include relevant ZenML documentation links. Prefer stable, high-level docs areas when the exact implementation path depends on the user's stack:

- Artifact management / external artifacts: `https://docs.zenml.io/user-guides/starter-guide/manage-artifacts`
- Dynamic pipelines: `https://docs.zenml.io/how-to/steps-pipelines/dynamic-pipelines`
- Scheduling: `https://docs.zenml.io/how-to/steps-pipelines/scheduling`
- Pipeline deployments / service-style triggering: `https://docs.zenml.io/how-to/deployment/deployment`
- ZenML Pro triggers: `https://docs.zenml.io/getting-started/zenml-pro/triggers`
- Orchestrators and scheduling: `https://docs.zenml.io/stacks/stack-components/orchestrators`
- Service connectors: `https://docs.zenml.io/stacks/service-connectors`
- Best practices / access management: `https://docs.zenml.io/user-guides/best-practices`

#### 3. Suggest installing the ZenML docs MCP server

> "For easier access to ZenML documentation while you work, you can install the ZenML docs MCP server: `claude mcp add zenmldocs --transport http https://docs.zenml.io/~gitbook/mcp`"

#### 4. Community support for unsupported patterns

When the migration has **2+ HIGH-severity flags**, generate a pre-made Slack message for `zenml.io/slack`. Include:
- what Dagster code is being migrated,
- the specific unsupported patterns,
- the workarounds already attempted, and
- what the user wants help deciding.

#### 5. Open GitHub issues for genuine feature gaps

When the migration reveals a genuine missing feature in ZenML -- not just "this works differently", but a real capability gap that multiple users would benefit from -- offer to open a GitHub issue on `zenml-io/zenml`.

#### 6. Run `/simplify` on the generated code

After migration is complete, always suggest running `/simplify` on the generated code to reduce migration noise, consolidate repetitive helper code, and make the result feel more like production code.

#### 7. Further customization via `zenml-pipeline-authoring`

Use `zenml-pipeline-authoring` for:
- Docker settings and remote execution
- YAML configuration
- custom materializers
- deployment and post-migration cleanup

## Important Behavioral Differences to Communicate

These are the most common sources of confusion after migration. Always mention the relevant ones in the migration report.

### Assets are not steps

A Dagster asset is a named data product with graph semantics around materialization, selection, partitions, and checks. A ZenML step is a unit of compute. The closest migration shape is usually:

- Dagster asset **compute body** -> ZenML `@step`
- Dagster asset **identity** -> artifact name or step output name
- Dagster asset **graph selection semantics** -> pipeline boundaries plus explicit source/loading patterns

### IO managers are not just materializers

If the original Dagster IO manager says, in effect, "when someone asks for this asset, go load table X from warehouse Y", then the real story is not serialization. The real story is data access logic. That logic usually belongs in a ZenML source/sink step, not only in a materializer.

### Partition keys are just the label, not the whole engine

Passing `partition_key="2026-04-07"` into a ZenML pipeline preserves the label. It does not automatically preserve partition mappings, backfills, freshness, or asset-reconciliation rules. Those must be rebuilt explicitly.

### Sensors become trigger systems, not steps that wait forever

A Dagster sensor is usually better reimagined as an external trigger or polling service. ZenML Pro also has schedule and platform-event trigger objects that attach to snapshots, which can fit some ZenML-platform event cases. They do not preserve Dagster sensor cursors, asset-sensor evaluation loops, or declarative automation semantics by themselves. Otherwise you risk turning a lightweight orchestration rule into an expensive long-running container.

## Anti-Patterns in Migration

| Anti-pattern | Why it is wrong | What to do instead |
|---|---|---|
| Treating every asset as its own pipeline | Destroys meaningful execution grouping | Group assets by real operational boundary |
| Forcing the entire asset graph into one pipeline | Hides the loss of subset materialization semantics | Split into multiple pipelines when needed |
| Translating every IOManager into a materializer | Loses business/data-access behavior | Separate serialization from explicit source/sink logic |
| Replacing sensors with infinite polling steps | Burns compute and changes operational behavior | Use ZenML Pro snapshot triggers where supported, external triggers, or bounded polling logic |
| Collapsing partition logic into a single untyped string without documenting the loss | Drops critical orchestration semantics | Preserve partition parameters explicitly and document gaps |
| Treating asset checks as comments instead of executable validation | Loses enforcement | Create validation steps and log metadata |

## References

### Detailed reference files

- [references/concept-map.md](references/concept-map.md) -- Full concept mapping tables and scheduling support matrix
- [references/code-patterns.md](references/code-patterns.md) -- Side-by-side code translations for the major Dagster patterns
- [references/gaps-and-flags.md](references/gaps-and-flags.md) -- Must-flag patterns, behavioral differences, and the migration decision tree

### Product documentation

- Dagster docs: `https://docs.dagster.io/`
- ZenML docs: `https://docs.zenml.io/`
