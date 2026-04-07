---
name: kedro-to-zenml-migration
description: >-
  Migrate Kedro pipelines and projects to idiomatic ZenML pipelines. Handles
  concept mapping (node->step, Pipeline->pipeline, Data Catalog->explicit
  boundary steps plus artifacts, params:->typed parameters), catalog analysis,
  code translation, hooks/runners/deployment mapping, and flags unsupported
  patterns (transcoding, dataset lifecycle hooks, namespace remapping,
  SharedMemoryDataset, slicing semantics) for human review. Use this skill
  whenever the user mentions Kedro migration, converting a Kedro project to
  ZenML, porting Kedro pipelines, replacing Kedro orchestration or deployment
  plugins with ZenML, or asks how a Kedro concept maps to ZenML -- even if they
  do not explicitly say "migrate". Also use when the user pastes `catalog.yml`,
  `parameters.yml`, `pipeline_registry.py`, node code, hook code, or describes
  a workflow using Kedro terminology such as node, pipeline, Data Catalog,
  `params:`, namespace, modular pipeline, runner, `MemoryDataset`, or
  transcoding in a ZenML context. If the user just asks a quick conceptual
  question ("what is the ZenML equivalent of `MemoryDataset`?"), answer it
  directly from the concept map -- no need to run the full migration workflow.
---

# Migrate Kedro to ZenML

This skill translates Kedro projects into idiomatic ZenML pipelines. It handles the full migration workflow: analyze the Kedro project, classify each pattern as direct / approximate / absent, translate what maps cleanly, flag what needs redesign, and produce a working ZenML project plus a clear migration report.

## How migration works at a high level

Kedro and ZenML are both Python-first workflow systems, so the business logic inside node functions often survives migration surprisingly well. The hard part is not the node code. The hard part is the **contract around the node code**.

Kedro uses the **Data Catalog** as the central registry of datasets, storage details, credentials, versioning, and sometimes representation tricks such as transcoding. ZenML does **not** have a `DataCatalog` equivalent. ZenML treats internal handoffs as typed **artifacts** and expects external reads and writes to be made explicit in step code, materializers, stack settings, and secrets.

So this migration is not a rename exercise. It is mostly a careful rewrite of:

1. **Boundary handling** -- catalog-driven reads and writes become explicit loader/exporter steps
2. **Data flow** -- string dataset names become typed artifact edges
3. **Configuration and auth** -- `parameters.yml` and `credentials.yml` move into ZenML parameters, YAML config, secrets, and stack settings
4. **Operational semantics** -- runners, slicing, hooks, namespaces, and deployment plugins must be checked for behavior drift

### The three mapping types

Every Kedro concept falls into one of these categories:

| Type | Meaning | Action |
|------|---------|--------|
| **Direct** | Clean 1:1 mapping exists | Translate automatically |
| **Approximate** | Similar intent exists but semantics differ | Translate with caveats noted in the migration report |
| **Absent** | No ZenML equivalent exists | Flag for human review with redesign suggestions |

See [references/concept-map.md](references/concept-map.md) for the full tables.

## The Migration Workflow

### Phase 1: Receive and Analyze the Kedro Project

Ask the user for the Kedro project files. Read them before writing any code. Prefer this order because it reveals the real contract of the system:

1. `conf/base/catalog.yml` and any environment-specific catalog files
2. `conf/base/parameters.yml` and related parameter files
3. `conf/local/credentials.yml` references if present
4. node files (`nodes.py` and related modules)
5. pipeline files (`pipeline.py`, modular pipeline factories)
6. `pipeline_registry.py`
7. `settings.py`
8. custom dataset classes
9. hook implementations
10. runner or deployment usage (`kedro run`, Airflow/Kubeflow/Vertex/Docker plugins, CI entrypoints)

For each project, inventory the following **before** deciding how to migrate it:

1. **Catalog structure** -- Which datasets are external inputs, external outputs, intermediates, free runtime datasets, versioned datasets, credential-bound datasets, transcoded aliases, partitioned datasets, incremental datasets, or custom datasets?
2. **Node graph** -- Which nodes are pure transforms and which nodes are really IO boundaries hidden behind catalog entries?
3. **Parameters** -- Which inputs come from `params:` references? Are they simple scalars, nested config objects, or runtime overrides?
4. **Namespaces and modular pipelines** -- Is reuse relying on automatic remapping or namespace-prefixed dataset names?
5. **Hooks** -- Any `before_node_run`, `after_node_run`, `on_node_error`, dataset lifecycle hooks, command hooks, or catalog hooks?
6. **Execution habits** -- Does the team rely on `--from-nodes`, `--to-nodes`, `--tags`, `--from-inputs`, `--to-outputs`, or `--only-missing-outputs`?
7. **Runners** -- `SequentialRunner`, `ParallelRunner`, `ThreadRunner`, `SharedMemoryDataset`, or other memory-sensitive behavior?
8. **Deployment intent** -- Is Kedro being exported to Airflow, Kubeflow, Vertex AI, Docker, or paired with `kedro-mlflow` / Kedro-Viz?

### Phase 2: Classify and Plan

After Phase 1, classify everything as direct / approximate / absent using [references/concept-map.md](references/concept-map.md) and [references/gaps-and-flags.md](references/gaps-and-flags.md).

#### Quick classification guide

**Direct or nearly direct translations (usually safe to generate automatically):**
- Plain node function -> `@step`
- Multiple inputs / outputs -> step args + tuple returns with `Annotated[...]`
- `Pipeline` -> `@pipeline`
- `parameters.yml` + `params:` -> typed pipeline / step parameters
- Registry-organized project -> normal Python module layout

**Approximate translations (generate, but explain what changed):**
- `pandas.CSVDataset`, `ParquetDataset`, `JSONDataset`, `ExcelDataset` -> explicit loader/exporter boundary steps
- `versioned: true` -> ZenML artifact versioning, explicit external version handling, or both
- custom datasets -> loader/exporter logic and sometimes custom materializers
- `ParallelRunner` -> orchestrator-managed parallelism with isolated steps
- `kedro-mlflow` -> ZenML experiment tracker
- `kedro-docker` -> `DockerSettings` and stack-driven containerization
- deployment plugins -> ZenML orchestrators and stack components

**Absent / redesign required (must be flagged):**
- `DataCatalog` as a global dataset registry
- transcoding (`dataset@pandas`, `dataset@spark`)
- automatic namespace/remapping semantics
- `SharedMemoryDataset` / `SharedMemoryDataCatalog`
- dataset lifecycle hooks
- slicing semantics (`--from-nodes`, `--to-nodes`, `--only-missing-outputs`, etc.)
- dataset factories and catch-all pattern resolution

#### Present the migration plan before generating code

Before writing code, summarize your findings for the user:

> "Here's what I found in your Kedro project:
> - **Direct translations** (will migrate cleanly): [list]
> - **Approximate translations** (will work but with caveats): [list]
> - **Needs redesign** (cannot be auto-migrated safely): [list]
>
> The main migration theme is: [for example, 'catalog-driven IO becomes explicit boundary steps'].
> Shall I proceed with the migration?"

If there are HIGH-severity flags, explain each one concretely:
- what the Kedro project currently does
- why ZenML cannot preserve that behavior automatically
- what the safest redesign approach looks like

### Phase 3: Generate ZenML Code

Translate the Kedro project into a ZenML project. Follow these conventions strictly.

#### Project structure

Every migrated project MUST use this layout:

```text
migrated_pipeline/
├── steps/                    # One file per step
│   ├── load_customers.py
│   ├── transform_features.py
│   └── export_report.py
├── pipelines/
│   └── my_pipeline.py
├── materializers/            # Only when truly needed
├── configs/
│   ├── dev.yaml
│   └── prod.yaml
├── run.py                    # argparse, not click
├── README.md
└── pyproject.toml
```

Key rules:
- One step per file in `steps/`
- Keep the pipeline definition separate from execution
- `run.py` uses `argparse`
- `pyproject.toml` should use `requires-python = ">=3.12"` and `zenml>=0.94.1`
- Always generate `configs/dev.yaml` and `configs/prod.yaml`
- Always generate a `README.md`
- Run `zenml init` at the project root

#### Core translation rules

See [references/code-patterns.md](references/code-patterns.md) for the concrete side-by-side examples. Use these rules consistently:

1. **Pure nodes become steps**
   - Move the node body into a `@step`
   - Add explicit type hints to all inputs and outputs
   - Use `Annotated[...]` when stable output names matter

2. **Catalog-driven IO becomes boundary steps**
   - File reads, table reads, report exports, and service writes should be explicit
   - Do not keep catalog-name indirection for internal edges
   - Internal handoffs should usually be artifacts, not persisted files

3. **`params:` becomes explicit typed parameters**
   - Convert `params:threshold` into a pipeline or step parameter
   - Put defaults and environment-specific overrides in YAML config
   - Do not recreate an implicit global parameter object if step signatures can stay explicit

4. **Versioning must be decided, not assumed**
   - If Kedro used versioned files for reproducibility only, ZenML artifact versioning may be enough
   - If downstream systems rely on concrete versioned paths, keep explicit exporter logic
   - If both matter, implement both

5. **Hooks only map partially**
   - `@step(on_success=...)` and `@step(on_failure=...)` are only partial substitutes
   - Dataset lifecycle hooks almost always need to be rebuilt at explicit boundaries
   - Do not claim a missing before-hook exists when it does not

6. **Namespaces and modular pipelines become explicit composition**
   - Reuse the same step graph via helper functions or wrapper pipelines
   - Use explicit invocation IDs and parameters
   - Do not silently mimic Kedro namespace behavior with simple string prefixes

7. **Runners and deployment plugins become stack design**
   - Translate runner expectations into orchestrator choice, resource settings, and step operators
   - Translate Airflow/Kubeflow/Vertex/Docker plugin intent into stack configuration

#### Comment style in generated code

Keep migration comments short and actionable:
- Use `# Migration note:` for brief inline caveats
- Use `# TODO(migration):` for required manual follow-up

Do not hide major semantic differences in comments alone. They must also appear in the migration report.

#### Handling approximate translations

When the migration is approximate, explain the difference right at the point of use:

```python
@step
def load_orders(path: str) -> pd.DataFrame:
    # Migration note: Kedro previously loaded this dataset through catalog.yml
    # with credentials and versioning managed outside the node code. In ZenML
    # that boundary is explicit here and configured via parameters + secrets.
    return pd.read_csv(path)
```

#### Handling absent patterns

Never silently approximate patterns with no real ZenML equivalent. Instead:

1. Add a clear `# TODO(migration):` comment in generated code
2. Record it in `MIGRATION_REPORT.md`
3. Offer a redesign approach

```python
# TODO(migration): UNSUPPORTED -- this Kedro project relied on transcoding
# (`features@pandas` and `features@spark`) for the same logical dataset.
# ZenML has no equivalent hidden representation switch. Pick one canonical
# artifact representation and make conversions explicit in dedicated steps.
```

### Phase 4: Produce the Migration Report

After generating the ZenML project, produce a `MIGRATION_REPORT.md` in the project root:

```markdown
# Migration Report: [Kedro Project] -> [ZenML Pipeline]

## Summary
- **Source**: Kedro project `[project_name]`
- **Target**: ZenML pipeline `[pipeline_name]`
- **Nodes migrated**: X direct, Y approximate, Z flagged
- **Catalog entries analyzed**: N

## Direct Translations
| Kedro Concept | ZenML Equivalent | Notes |
|---|---|---|
| node(clean_orders) | steps/clean_orders.py | Pure transform |

## Approximate Translations
| Kedro Pattern | ZenML Equivalent | What Changed |
|---|---|---|
| pandas.CSVDataset | explicit loader/exporter steps | IO boundary is now explicit |
| versioned: true | artifact versioning + optional external export versioning | File-path semantics may differ |

## Flagged for Review
| Kedro Pattern | Severity | Issue | Suggested Redesign |
|---|---|---|---|
| transcoding | HIGH | No hidden representation switching in ZenML | Use explicit conversion steps |
| before_dataset_loaded hook | HIGH | No dataset lifecycle hook equivalent | Move logic into loader step |

## Catalog Translation Summary
| Catalog Entry | Original Type | Migration Target | Notes |
|---|---|---|---|
| raw_orders | pandas.CSVDataset | load_orders step | external input |
| model | pickle.PickleDataset | artifact + exporter step | check path/version semantics |

## Configuration Migration Summary
- `parameters.yml` keys moved to: [pipeline config path(s)]
- Runtime overrides previously passed through: [Kedro mechanism]
- New ZenML config entrypoints: [dev/prod yaml files]

## Credential / Auth Migration Summary
- `credentials.yml` entries moved to ZenML secrets / env vars / service connectors
- Any manual setup still required: [list]

## Namespace / Composition Summary
- Which modular pipelines were preserved
- Which namespaces/remappings required explicit wrapper logic

## Runner / Deployment Migration Summary
- Original runner(s): [SequentialRunner / ParallelRunner / ThreadRunner]
- Original plugins: [kedro-airflow / kedro-kubeflow / kedro-vertexai / kedro-docker / kedro-mlflow]
- New ZenML stack assumptions: [orchestrator / step operator / tracking setup]

## Limitations and Key Differences
[Put the most important behavior differences here BEFORE the benefits section]

## What's NOT Migrated
[List unsupported or intentionally deferred patterns]

## What You Get for Free After Migration
- Artifact versioning and lineage
- Step caching
- Stack abstraction
- Stronger typed artifact flow
- Optional dynamic pipelines and cross-pipeline artifact reuse
- Better alignment with experiment trackers, step operators, and deployment workflows

## Recommended Next Steps
1. Run `zenml-quick-wins`
2. Install the ZenML docs MCP server
3. Use `zenml-pipeline-authoring` for deeper Docker / materializer / YAML / deployment work
4. Follow up on every HIGH-severity flag before production use
```

### Phase 5: Suggest Next Steps

After migration is complete, always include a **Recommended Next Steps** section in the migration report **and** communicate it to the user directly.

#### 1. Run the `zenml-quick-wins` skill

Always suggest this first:

> "Now that the migration is done, I recommend running the `zenml-quick-wins` skill to add metadata logging, experiment tracking, alerters, and other production features."

#### 2. Include documentation links for flagged patterns

For every flagged pattern, include relevant ZenML documentation links when they help:

- YAML configuration: `https://docs.zenml.io/concepts/steps_and_pipelines/yaml_configuration`
- Secrets: `https://docs.zenml.io/concepts/secrets`
- Dynamic pipelines: `https://docs.zenml.io/concepts/steps_and_pipelines/dynamic_pipelines`
- Containerization: `https://docs.zenml.io/concepts/containerization`
- Step operators: `https://docs.zenml.io/stacks/stack-components/step-operators/custom`

#### 3. Suggest installing the ZenML docs MCP server

> "For easier access to ZenML docs while you keep iterating, you can install the ZenML docs MCP server: `claude mcp add zenmldocs --transport http https://docs.zenml.io/~gitbook/mcp`"

#### 4. Community support for unsupported patterns

When there are HIGH-severity flags, offer to help the user ask the ZenML community for guidance. **When there are 2+ HIGH-severity flags**, generate a ready-to-send Slack message for `zenml.io/slack` that includes:
- what Kedro project is being migrated
- the exact unsupported patterns
- what workarounds were suggested
- what behavior might change if the workaround is used

#### 5. Open GitHub issues for genuine product gaps

When the migration reveals a real missing ZenML capability -- not merely a different design, but a genuine gap that multiple users would benefit from -- offer to open a GitHub issue on `zenml-io/zenml`.

#### 6. Suggest `/simplify`

After migration, always suggest running `/simplify` on the generated code so migration comments, wrappers, and duplication can be cleaned up.

#### 7. Further customization via `zenml-pipeline-authoring`

Recommend `zenml-pipeline-authoring` when the user needs deeper help with:
- `DockerSettings`
- YAML configuration
- custom materializers
- step operators
- deployments
- multi-environment architecture

## Important Behavioral Differences to Communicate

Always mention the relevant behavior differences in the migration report.

### Data Catalog != Artifact Store

Kedro uses a named dataset registry that hides storage details from node code. ZenML uses typed artifacts flowing between steps. That means the migration often adds explicit loader/exporter steps at the edges of the graph.

### `MemoryDataset` != "stay in memory"

Kedro can keep intermediates ephemeral. ZenML artifacts are normally persisted and versioned. If ephemerality mattered for correctness or cost, flag it.

### Namespaces != simple prefixes

Kedro namespaces and remapping change how datasets and parameters resolve. ZenML reuse is explicit Python composition. These are related ideas, not the same feature.

### Slicing != caching

Kedro's CLI slicing changes which part of a graph runs. ZenML caching reuses outputs when inputs, code, and settings match. Do not present them as equivalents.

### Runners != orchestrators

Kedro runners are not a direct code-level concept in ZenML. Translate them into stack design, step isolation assumptions, and resource settings.

## Anti-Patterns to Avoid

- Do **not** recreate a fake `DataCatalog` abstraction on top of ZenML unless the user explicitly wants a transitional adapter and understands the tradeoff.
- Do **not** convert every Kedro dataset into a custom materializer. Many Kedro datasets are storage adapters, not artifact types.
- Do **not** silently flatten transcoding into one representation.
- Do **not** claim dataset lifecycle hooks or namespace remapping "basically work the same".
- Do **not** force a full repo layout rewrite before behavior is correct.

## Additional References

- [references/concept-map.md](references/concept-map.md) -- full Kedro -> ZenML concept map
- [references/code-patterns.md](references/code-patterns.md) -- side-by-side translation examples
- [references/gaps-and-flags.md](references/gaps-and-flags.md) -- must-flag patterns, behavioral differences, and decision tree
