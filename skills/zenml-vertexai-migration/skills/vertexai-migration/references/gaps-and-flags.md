# Gaps, Flags, and Behavioral Differences

This reference covers the places where a naive Vertex / KFP -> ZenML translation silently changes behavior. These are the boundaries the migration skill must not cross without explicitly telling the user.

## Table of Contents

- [Must-Flag Patterns](#must-flag-patterns)
- [Artifact Contract Classification Guide](#artifact-contract-classification-guide)
- [Behavioral Differences](#behavioral-differences)
- [Migration Decision Tree](#migration-decision-tree)
- [ZenML Features with No Vertex Equivalent](#zenml-features-with-no-vertex-equivalent)

---

## Must-Flag Patterns

These patterns must **never** be silently approximated.

### GCPC operators

**What Vertex / KFP does**: GCPC gives you prebuilt KFP components for BigQuery, AutoML, Model Upload, Endpoint Deploy, Batch Prediction, Dataflow, Dataproc, and more.

**Why it matters**: These are not just helper functions. They are part of the KFP / Vertex authoring layer.

**ZenML gap**: ZenML has no GCPC-style operator catalog.

**Safe redesign**: Rewrite each GCPC node as an ordinary ZenML step that calls the relevant Google Cloud SDK or API directly.

### Path- and URI-coupled artifact contracts

**What Vertex / KFP does**: Components commonly use `.path`, `.uri`, `InputPath`, and `OutputPath` as real runtime contracts.

**Why it matters**: If you flatten that contract into a DataFrame or dict blindly, you may erase directory layout, sibling files, or location identity.

**ZenML gap**: ZenML’s default contract is Python values plus materializers.

**Safe redesign**: First classify the component as value-centric, location-aware, or template/platform-coupled. Use custom materializers or explicit URI contracts where needed.

### Compiled templates and template registries

**What Vertex / KFP does**: Pipelines can be compiled, uploaded, versioned, and scheduled from a template path.

**Why it matters**: Some teams are really using a template distribution workflow, not just a pipeline definition.

**ZenML gap**: ZenML does not have a 1:1 "compiled template registry" concept.

**Safe redesign**: Prefer a ZenML-native rewrite. If templates must stay, wrap the existing `PipelineJob` submission in a single ZenML step and call it a partial migration.

### Runtime branching semantics

**What Vertex / KFP does**: `dsl.If` / `dsl.Condition` can branch on upstream outputs at runtime.

**Why it matters**: Turning that into a static Python `if` changes the graph and the execution story.

**ZenML gap**: Runtime branching requires dynamic pipelines and `.load()`, not static graph construction.

**Safe redesign**: Use `@pipeline(dynamic=True)` and document the semantic change explicitly.

### `dsl.ParallelFor` and `dsl.Collected`

**What Vertex / KFP does**: fans out work at the orchestration layer and later fans it back in.

**Why it matters**: Replacing it with a plain `for` loop serializes work and changes observability and retry semantics.

**ZenML gap**: `.map()` is the closest match, but not identical.

**Safe redesign**: Use dynamic pipelines and `.map()` only when the semantics are good enough; otherwise redesign the workflow.

### `dsl.ExitHandler`

**What Vertex / KFP does**: provides exit-task behavior after a scoped block, often for cleanup or notifications.

**Why it matters**: Users often rely on this for "always run" behavior even on failure.

**ZenML gap**: No exact equivalent.

**Safe redesign**: Use idempotent cleanup, hooks, or external failure handling. Never tell the user this is a direct translation.

### Vertex schedule lifecycle and concurrency knobs

**What Vertex / KFP does**: native schedule creation includes extra lifecycle and concurrency controls.

**Why it matters**: Teams may depend on those knobs operationally.

**ZenML gap**: `Schedule(...)` exposes only the common subset.

**Safe redesign**: Migrate the cron/start/end parts and document the rest as manual follow-up.

### Vertex Experiments / Model Registry assumptions

**What Vertex / KFP does**: experiment association and model resources are explicit platform objects.

**Why it matters**: Users may think those features are preserved automatically once the code compiles.

**ZenML gap**: Similar intent, different resource model.

**Safe redesign**: Add explicit tracker wiring and explicit upload / deploy steps where needed.

---

## Artifact Contract Classification Guide

Treat this the way the Databricks skill treats notebook triage: it is the first risk-sorting step.

### Detection checklist

Look for these signals in each component:

- uses `.path`
- uses `.uri`
- uses `InputPath(...)` / `OutputPath(...)`
- writes sibling files next to the artifact path
- passes the artifact path into an external binary / container / SDK
- uses importer components
- returns only scalars, small dicts, or normal Python objects

### Classification outcomes

| Classification | Signs | Migration stance |
|---|---|---|
| **Auto-translatable (value-centric)** | Uses artifacts as data values; `.path` is just a serialization boundary | Convert to normal Python inputs/returns and let ZenML materialize them |
| **Semi-automatic (location-aware)** | Uses `.path` / `.uri` meaningfully, but the contract is still understandable | Keep an explicit reference type, `Path`, URI object, or custom materializer |
| **Manual redesign required (template / platform / path coupled)** | Relies on placeholders, compiled template behavior, directory conventions, or opaque runtime injection | Stop and redesign before claiming a migration |

### Examples

| Pattern | Classification | Why |
|---|---|---|
| reads CSV from `dataset.path`, then returns a trained sklearn model | auto-translatable | The file path is only a serialization boundary |
| passes `model.uri` into a downstream deployment helper | semi-automatic | The location identity matters |
| writes multiple artifacts into a KFP-managed folder structure and expects downstream placeholder substitution | manual redesign required | The contract is tied to KFP runtime behavior |

---

## Behavioral Differences

These are the differences that are often safe to migrate, but unsafe to hide.

### Artifact model

| Aspect | Vertex / KFP | ZenML |
|---|---|---|
| primary unit | runtime-managed artifact reference | materialized Python value |
| common access pattern | `.path`, `.uri`, metadata helpers | typed objects + materializers |
| naming | artifact classes and pipeline metadata | `Annotated[..., "name"]`, artifact metadata, materializers |
| large-file handling | explicit file/URI contract | artifact store + materializer contract |

### Pipeline execution / compile model

| Aspect | Vertex / KFP | ZenML |
|---|---|---|
| authoring output | compiled template | Python pipeline definition |
| submission model | `PipelineJob(template_path=...)` | pipeline execution through active stack |
| template lifecycle | first-class | no direct equivalent |
| portability story | tied to KFP / Vertex authoring | stack abstraction across backends |

### Control flow

| Aspect | Vertex / KFP | ZenML |
|---|---|---|
| runtime branching | built into KFP DSL / backend | dynamic pipelines with `.load()` |
| fan-out | `dsl.ParallelFor` | `.map()` |
| fan-in | `dsl.Collected` | reducer step over mapped outputs |
| exit handling | `dsl.ExitHandler` | no exact equivalent |

### Resource management

| Aspect | Vertex / KFP | ZenML |
|---|---|---|
| portable resources | limited | `ResourceSettings(...)` |
| backend-specific knobs | task / job settings | orchestrator-specific settings |
| images / packages | component decorators | `DockerSettings(...)` and build config |
| network / service account / persistent resources | Vertex job config | Vertex orchestrator configuration |

### Scheduling

| Aspect | Vertex / KFP | ZenML |
|---|---|---|
| schedule creation | `PipelineJob.create_schedule(...)` | `Schedule(...)` |
| supported knobs | richer lifecycle / concurrency surface | common subset only |
| schedule management | Vertex APIs / UI | partial orchestration-layer support |
| template dependency | often compiled-template based | not part of ZenML model |

### Monitoring, experiments, and metadata

| Aspect | Vertex / KFP | ZenML |
|---|---|---|
| experiments | Vertex Experiments | experiment tracker stack component |
| model resources | Vertex Model Registry | ZenML model / artifact control-plane concepts |
| metadata | Vertex ML Metadata | ZenML metadata store + artifact lineage |
| run UI | Vertex Pipeline UI | ZenML dashboard plus orchestrator metadata links |

---

## Migration Decision Tree

Use this to classify each node before rewriting it:

```text
1) Is this node a GCPC operator?
   ├── Yes -> classify as absent -> rewrite as SDK-calling step
   └── No -> continue

2) Is it a container component?
   ├── Yes -> classify as approximate -> inspect command/args/image contract carefully
   └── No -> continue

3) Is it a plain Python component?
   ├── Yes -> continue
   └── No -> analyze as a custom platform pattern

4) What is the artifact contract?
   ├── value-centric -> normal typed inputs/returns
   ├── location-aware -> explicit URI/path contract or custom materializer
   └── template/platform/path-coupled -> manual redesign required

5) Does control flow depend on upstream outputs?
   ├── Yes -> dynamic pipeline design required
   └── No -> static pipeline may be fine

6) Does the pipeline rely on dsl.If / dsl.Condition / dsl.ParallelFor / dsl.Collected?
   ├── Yes -> record as approximate and document the semantic change
   └── No -> continue

7) Does the workflow rely on dsl.ExitHandler or final-status cleanup?
   ├── Yes -> classify as absent -> redesign cleanup / notifications
   └── No -> continue

8) Does the user need template-based submission to remain?
   ├── Yes -> partial migration via black-box PipelineJob submission step
   └── No -> full ZenML-native rewrite
```

---

## ZenML Features with No Vertex Equivalent

These are useful "what you get for free" items to mention in migration reports.

| Feature | Value |
|---|---|
| **stack abstraction** | same pipeline code can move across stacks without rewriting the workflow |
| **portable resource settings** | resource intent is expressed separately from backend-specific details |
| **materializers** | explicit, reusable serialization logic for domain-specific Python types |
| **artifact versioning and lineage** | richer artifact tracking as a first-class concept |
| **step caching** | rerun only what changed |
| **model control plane** | explicit model lifecycle capabilities in the ZenML ecosystem |
| **service connectors** | unified auth abstraction for cloud services |
| **snapshots and stack-aware runs** | a different reproducibility story from compiled templates |
