# Gaps, Flags, and Behavioral Differences

This reference covers the patterns that are most dangerous to silently approximate during Kedro -> ZenML migration: the places where Kedro and ZenML behave differently enough that a naive translation changes the real behavior of the pipeline.

## Table of Contents

- [Must-Flag Patterns](#must-flag-patterns)
- [Behavioral Differences](#behavioral-differences)
- [Migration Decision Tree](#migration-decision-tree)
- [ZenML Features with No Kedro Equivalent](#zenml-features-with-no-kedro-equivalent)
- [Anti-Patterns](#anti-patterns)

---

## Must-Flag Patterns

These patterns must **never** be silently approximated. Flag them in the migration report and require human review.

| Pattern | What Kedro does | Why it matters | ZenML gap | Redesign approaches |
|---|---|---|---|---|
| Data Catalog abstraction | Nodes are pure functions over dataset names resolved by `DataCatalog` | It hides IO location, type, auth, and versioning from node code | No `DataCatalog` equivalent | Convert external boundaries into loader/exporter steps, keep internal edges as artifacts, move config into YAML + stack settings + secrets |
| Transcoding | Same logical dataset can be loaded as `dataset@pandas`, `dataset@spark`, etc. | Different consumers can see different concrete representations of the same data | No equivalent | Choose one canonical artifact representation and make conversions explicit in dedicated steps |
| `MemoryDataset` and free datasets | Intermediates may be undeclared and ephemeral | Migration can accidentally turn ephemeral state into persisted artifacts | Closest match persists by default | Use normal step outputs, but explicitly acknowledge the persistence change and redesign if ephemerality mattered |
| `SharedMemoryDataset` / `SharedMemoryDataCatalog` | Supports in-memory sharing for `ParallelRunner` | Shared-memory assumptions can fail badly on remote or isolated execution | No equivalent | Redesign around artifact passing, external stores, or explicit batching |
| Credential injection via `credentials.yml` | Dataset auth is declared outside node code and referenced by dataset name | Auth becomes invisible coupling between catalog and runtime | No dataset-local auth hook | Move to ZenML secrets, env vars, service connectors, or explicit step-local auth |
| Modular pipeline namespacing | Automatically prefixes dataset names and supports remapping | Reuse semantics depend on namespace resolution, not just labels | No equivalent | Replace with wrapper functions, explicit invocation IDs, and explicit parameter / boundary naming |
| Pipeline slicing | `--from-nodes`, `--to-nodes`, `--tags`, `--only-missing-outputs` change runtime execution of the graph | Teams often rely on these for partial reruns and backfills | No equivalent | Split monolithic pipelines, promote reusable artifacts, create dedicated entrypoint pipelines |
| `ParallelRunner` / `ThreadRunner` semantics | In-process concurrency with specific memory and Spark caveats | Migration can silently change memory-sharing and failure behavior | Only approximate via orchestrators | Re-benchmark branch structure, resource settings, data movement, and step isolation |
| Dataset lifecycle hooks | `before_dataset_loaded`, `after_dataset_saved`, etc. | Validation, auditing, masking, or side effects may live here | No equivalent | Recreate logic in loader/exporter steps, custom materializers, or external platform hooks |
| Dataset factories and catch-all patterns | Catalog patterns can synthesize datasets from names | A lot of "magic" may live in naming schemes | No equivalent | Expand into explicit boundaries or build your own thin config layer consciously |
| Strict Kedro project structure | `conf/`, `src/`, `data/`, registry conventions | Teams may rely on these conventions for tooling and onboarding | ZenML is flexible | Keep structure temporarily if it lowers risk, then simplify later |

---

## Behavioral Differences

### Data Catalog vs artifact store and materializers

| Topic | Kedro | ZenML | Migration consequence |
|---|---|---|---|
| Primary abstraction | Named datasets in a catalog | Typed artifacts in an artifact store | Start migration from `catalog.yml`, then rewrite boundaries explicitly |
| Internal edge meaning | Dataset name resolved at runtime | Artifact produced by an upstream step | Internal names stop being storage aliases and become lineage artifacts |
| IO declaration | Declarative in catalog metadata | Usually explicit in step code or stack settings | Expect more boundary code in ZenML |
| Serialization | Dataset implementation handles load/save | Materializer handles serialization based on Python type | Custom Kedro datasets often split into loader code plus optional materializer work |
| Versioning | Versioned under dataset path | Versioned as artifacts with lineage | File-version semantics and artifact-version semantics are not interchangeable |

### Configuration model

| Topic | Kedro | ZenML | Migration consequence |
|---|---|---|---|
| Data config | `catalog.yml` | No single equivalent | Config fans out into steps, materializers, YAML, secrets, and stack settings |
| Parameters | `parameters.yml` + `params:` | Pipeline / step parameters plus YAML config | Friendly migration, but wiring changes |
| Secrets | `credentials.yml` | ZenML secrets, env vars, service connectors | Auth moves out of dataset definitions |
| Loader semantics | OmegaConfigLoader and resolvers | ZenML YAML config hierarchy | If you depend on OmegaConf behavior, you may need a custom layer |

### Project structure

| Topic | Kedro | ZenML | Migration consequence |
|---|---|---|---|
| Repository layout | Opinionated, starter-driven layout | Flexible layout | Keep the old tree during the first pass if it reduces migration risk |
| Pipeline discovery | Registry and `__default__` | Explicit pipeline functions | Remove reliance on registry conventions |
| Scaffolding | `kedro new`, `kedro pipeline create` | No core equivalent | Project scaffolding becomes your internal tooling choice |

### Hooks

| Topic | Kedro | ZenML | Migration consequence |
|---|---|---|---|
| Hook granularity | Context, catalog, pipeline, node, dataset, command | Step and pipeline success/failure hooks | Hook-heavy Kedro code almost always needs redesign |
| Before-step hooks | Yes | No | Put preconditions and validation in explicit steps or wrapper code |
| Dataset lifecycle hooks | Yes | No | Move these concerns into boundary steps or custom materializers |

### Runners vs orchestrators

| Topic | Kedro | ZenML | Migration consequence |
|---|---|---|---|
| Execution control | Runner chooses sequential / process / thread behavior | Active stack and orchestrator choose runtime behavior | Translate runner assumptions into stack design, not just code |
| Parallelism | Often same-machine concurrency | Often isolated remote step execution | Revisit memory, data transfer, and failure assumptions |
| Scheduling | Usually via external platform plugins | Native schedule abstraction on supported orchestrators | Scheduling becomes part of deployment rather than a Kedro export plugin |

### Testing philosophy

| Topic | Kedro | ZenML | Migration consequence |
|---|---|---|---|
| Unit-test sweet spot | Pure node functions and catalog-backed integration tests | Pure step functions and artifact-backed integration tests | Most business logic tests survive almost unchanged |
| Mocking strategy | Swap catalog entries, `MemoryDataset`, test catalogs | Test pure step bodies or small pipelines and inspect artifacts | Catalog-mocking tests often become loader/exporter-step tests |
| Cross-pipeline reuse tests | Namespaces and remapping behavior matter | Artifact lineage and explicit wrapper pipelines matter | Tests need to move from "dataset names resolve correctly" to "artifacts and boundaries connect correctly" |

---

## Migration Decision Tree

A reliable Kedro migration starts from the catalog, then moves outward.

```text
INPUT: catalog entries, parameters, nodes, pipelines, hooks, runners, deployment plugins

STEP 1: CLASSIFY CATALOG ENTRIES
  For each entry in catalog.yml:
    - Is it an external input?
    - Is it an external output?
    - Is it an intermediate dataset?
    - Is it a free runtime dataset / MemoryDataset?
    - Is it versioned?
    - Does it use credentials: ?
    - Does it use transcoding?
    - Is it partitioned / incremental / custom?

STEP 2: DECIDE BOUNDARY VS INTERNAL EDGE
  IF external input:
    create explicit loader step
    consider ExternalArtifact only if pre-existing data should enter the lineage graph directly
  ELSE IF external output:
    create explicit exporter step
  ELSE:
    prefer ordinary step output artifacts

STEP 3: RESOLVE VERSIONING
  IF versioned behavior was mainly for reproducibility:
    use artifact versioning
  ELSE IF downstream systems depend on concrete versioned paths:
    keep explicit exporter logic
  ELSE:
    use both and document the split

STEP 4: RESOLVE CREDENTIALS
  IF catalog entry uses credentials:
    move auth to secrets / env vars / service connectors / stack config
    do not recreate dataset-local hidden auth

STEP 5: TRANSLATE NODES
  IF node is a pure transform:
    convert to @step with typed inputs/outputs
  ELSE IF node mainly performs boundary IO:
    keep it as a dedicated loader/exporter step

STEP 6: TRANSLATE PIPELINES AND COMPOSITION
  IF namespaces or remapping are essential:
    rebuild using wrapper functions, explicit invocation IDs, and explicit parameter wiring
  ELSE:
    compose steps directly in @pipeline functions

STEP 7: HANDLE CLI SLICING HABITS
  IF team depends on --from-nodes / --to-nodes / --only-missing-outputs / --tags:
    design smaller entrypoint pipelines, artifact-reuse pipelines, or backfill-specific entrypoints
    do not claim caching is equivalent

STEP 8: HANDLE HOOKS
  IF success/failure hook:
    maybe map to step/pipeline hooks
  ELSE IF dataset lifecycle hook:
    rebuild explicitly in boundary steps or materializers

STEP 9: HANDLE RUNNERS
  Translate SequentialRunner / ParallelRunner / ThreadRunner into:
    - orchestrator choice
    - resource settings
    - data movement design
    - concurrency and isolation assumptions

STEP 10: HANDLE DEPLOYMENT PLUGINS
  Replace "export Kedro to platform X" with "run ZenML on stack X"
  Move:
    container behavior -> DockerSettings
    resources -> ResourceSettings
    schedules -> Schedule
    tracking -> experiment tracker stack component
```

### Human-review checkpoints

Stop and ask for confirmation when any of the following are true:

- More than one HIGH-severity flag appears in the same pipeline
- The project relies on transcoding
- Namespace/remapping is central to reuse
- Dataset lifecycle hooks contain business-critical logic
- `SharedMemoryDataset` or shared-memory assumptions appear
- The team depends heavily on slicing for production operations

---

## ZenML Features with No Kedro Equivalent

These are worth calling out because migration can improve the system, not just reproduce it.

| ZenML capability | What the user gains after migration | Notes |
|---|---|---|
| Stack abstraction | Same pipeline code can run against different orchestrators, artifact stores, trackers, and registries by switching stacks | Kedro plugins are usually platform-specific |
| Artifact lineage | First-class traceability of step outputs | Kedro versioning is dataset-centric, not artifact-graph-centric |
| Step caching | Skip re-execution when code, inputs, and settings match | Not the same as Kedro slicing |
| External artifacts | Register pre-existing data as artifacts | Useful for bridging non-ZenML-produced data into the lineage graph |
| Dynamic pipelines | Runtime-shaped DAGs for some fan-out / branching patterns | More flexible than static modular composition, but still not a universal replacement |
| Step operators | Run specific steps in specialized environments | More granular than "run the whole project on platform X" |
| Model Control Plane | Model versioning and promotion workflows | Goes beyond Kedro core and most Kedro plugins |
| Rich metadata logging | Stronger artifact- and run-level metadata model | Good follow-up after the structural migration is stable |

---

## Anti-Patterns

| Anti-pattern | Why it is wrong | What to do instead |
|---|---|---|
| Recreating a fake `DataCatalog` on top of ZenML | Preserves the old mental model and hides the real artifact contract | Use explicit loader/exporter steps and artifact edges |
| Treating `catalog.yml` as if it maps to one ZenML YAML file | The information fans out across steps, materializers, secrets, and stack settings | Split responsibilities intentionally |
| Converting every Kedro dataset into a materializer | Many Kedro datasets are storage adapters, not artifact types | Use boundary steps for storage adapters; reserve materializers for true serialization concerns |
| Silently flattening transcoding into one representation | Changes behavior without telling the user | Choose a canonical representation and add explicit conversion steps |
| Treating caching as a substitute for slicing | Caching and slicing solve different problems | Build smaller pipelines or artifact-reuse entrypoints |
| Porting `credentials.yml` into checked-in ZenML config | Weakens secret hygiene and misses ZenML's auth model | Use ZenML secrets, env vars, or service connectors |
| Assuming `ParallelRunner` means "ZenML runs in parallel too" | The isolation and memory model are different | Re-evaluate branch structure, data size, and remote execution costs |
| Forcing a full repo rewrite before semantics are correct | Adds churn while behavior is still moving | Keep the old Kedro layout temporarily if it reduces risk, then simplify later |
