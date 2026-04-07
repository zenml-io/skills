# Gaps, Flags, and Behavioral Differences

This reference covers the Dagster patterns that are most dangerous to silently approximate during migration -- the places where Dagster and ZenML behave fundamentally differently, and where a naive translation changes the pipeline's actual behavior.

## Table of Contents

- [Must-Flag Patterns](#must-flag-patterns)
- [Behavioral Differences](#behavioral-differences)
- [Migration Decision Tree](#migration-decision-tree)
- [ZenML Features with No Dagster Equivalent](#zenml-features-with-no-dagster-equivalent)

---

## Must-Flag Patterns

These patterns must **never** be silently approximated. Flag them in the migration report and require human review.

### Asset selection and partial materialization

**What Dagster does**: Dagster treats assets as first-class addressable nodes. Users can materialize a subset of an asset graph, re-materialize one downstream asset while loading upstream assets from storage, or define jobs around subsets.

**Why it matters**: This is the biggest semantic mismatch in a Dagster -> ZenML migration. ZenML runs pipelines, not selected asset subsets.

**ZenML gap**: No first-class asset selection or partial materialization model.

**Redesign approaches**:
- Split the Dagster graph into multiple ZenML pipelines that match real operational boundaries.
- Use explicit source-loading steps or `ExternalArtifact` when a downstream pipeline needs previously produced data.
- Add a pipeline-boundary section to the migration report explaining what was split and why.

### Partitions, partition mappings, and backfills

**What Dagster does**: Partitions are built into asset definitions and jobs. Dagster can reason about partition keys, upstream/downstream partition mappings, and backfill ranges.

**Why it matters**: Partition-aware orchestration changes both execution and correctness. A naive translation to a single `run_date` parameter often drops important semantics.

**ZenML gap**: No first-class partition graph or partition-mapping engine.

**Redesign approaches**:
- Model the partition key explicitly as a pipeline parameter.
- Move backfill orchestration into external triggering logic or batch job wrappers.
- If the Dagster code depends on non-trivial partition mappings, classify it as HIGH severity and require a custom redesign.

### Declarative Automation / freshness policies / auto-materialize behavior

**What Dagster does**: Dagster can declaratively describe when assets should be materialized or considered stale, and can automate asset runs from those policies.

**Why it matters**: These policies are not just schedules; they are data-aware orchestration rules.

**ZenML gap**: No equivalent policy engine in OSS ZenML.

**Redesign approaches**:
- Re-express the intent as schedules, monitoring checks, and external triggers.
- Document what signal was lost (freshness deadline, dependency-aware automation, etc.).
- Keep the decision explicit in the migration report; do not quietly downgrade policy-driven automation to a plain cron job.

### Sensors and asset sensors

**What Dagster does**: Sensors poll external systems or Dagster state and emit runs, often with cursor semantics to avoid duplicate work.

**Why it matters**: Sensors encode event-driven orchestration and stateful polling logic.

**ZenML gap**: No sensor primitive, no cursor API, no lightweight deferral loop.

**Redesign approaches**:
- Use external eventing (webhooks, queues, cloud events) to trigger pipeline runs.
- If polling must remain, implement a polling service outside the ZenML pipeline or a polling step with strict timeout/resource notes.
- Treat sensor cursor behavior as a migration concern, not an implementation detail.

### IO managers with business logic

**What Dagster does**: IO managers can decide how outputs are written and how downstream consumers load them. In many codebases, they also embed warehouse/table conventions, lazy loading, or environment-specific routing.

**Why it matters**: If you translate a smart IO manager to a plain ZenML materializer, you often preserve serialization but lose the real data-access behavior.

**ZenML gap**: Materializers are not a drop-in replacement for arbitrary IO-manager behavior.

**Redesign approaches**:
- Keep serialization concerns in materializers.
- Move external read/write behavior into explicit source/sink steps.
- Flag any IO manager that does more than file/object serialization.

### Multi-asset subset semantics

**What Dagster does**: `@multi_asset` can expose several assets from one compute body, and some setups rely on selectively materializing only a subset.

**Why it matters**: A multi-output ZenML step can preserve the compute body, but not the independently materializable asset-subset semantics.

**ZenML gap**: No first-class subset selection for step outputs.

**Redesign approaches**:
- If all outputs are always produced together, a multi-output step is fine.
- If the code relies on subset execution, split the compute body or flag it as unsupported.

### Asset checks as independently managed graph nodes

**What Dagster does**: Asset checks are first-class checks attached to assets, can be executed independently, and participate in the asset graph view.

**Why it matters**: Validation remains important after migration, but the operational model changes.

**ZenML gap**: Checks are best represented as validation steps or metadata logging, not as independently selectable asset-graph nodes.

**Redesign approaches**:
- Translate the check logic into a validation step.
- Log the result as metadata and fail the step when appropriate.
- Flag any workflow that depends on independently running the checks outside the compute path.

### Observable source assets

**What Dagster does**: Dagster can observe external sources and attach metadata/freshness information to them as graph nodes.

**Why it matters**: This mixes data observability with orchestration.

**ZenML gap**: No first-class observable-source abstraction.

**Redesign approaches**:
- Model the external source as an `ExternalArtifact` or explicit ingestion step.
- Add a validation/observation step for metadata collection.
- Document the loss of asset-graph observability semantics.

---

## Behavioral Differences

These are the semantic differences that are safe to translate but important to communicate to the user.

### Asset materialization vs pipeline runs

| Aspect | Dagster | ZenML |
|---|---|---|
| **Primary execution unit** | Asset or job materialization | Pipeline run |
| **Addressability** | Individual assets and asset subsets | Entire pipeline entry point |
| **Re-use of upstream results** | Load upstream asset from storage without recompute | Use previously stored artifact via explicit source/loading pattern |
| **Primary mental model** | Data products in an asset graph | Typed compute steps in a pipeline DAG |

**Practical impact**: The same compute logic may migrate cleanly while the operational boundary changes dramatically. Always explain where a former Dagster graph was split into multiple ZenML pipelines.

### IO managers vs materializers + artifact store

| Aspect | Dagster IOManager | ZenML Materializer + Artifact Store |
|---|---|---|
| **Responsibility** | Store outputs and load downstream inputs | Serialize and load artifacts stored by ZenML |
| **External system reads** | Often embedded in the IO manager | Usually explicit step logic |
| **Control over downstream loading** | First-class | Limited to artifact materialization |
| **Good fit for business logic** | Common but risky | Better moved into explicit steps |

**Practical impact**: If a Dagster IO manager hides warehouse/table loading conventions, preserve that logic explicitly rather than assuming a custom materializer is sufficient.

### Resources vs stack components / connectors / secrets

| Aspect | Dagster | ZenML |
|---|---|---|
| **Injection model** | Resource dependency injection | Stack components, service connectors, secrets, or local helper construction |
| **Scope** | Often both infra-wide and per-call | Split by responsibility |
| **Best migration strategy** | Map by intent | Ask: is this infra, auth, or plain Python helper code? |

**Practical impact**: Treat each `ConfigurableResource` like a suitcase you unpack: credentials go one place, cluster settings another, and request-specific helper logic stays in code.

### Partition-aware execution vs parameterized runs

| Aspect | Dagster | ZenML |
|---|---|---|
| **Partition key** | First-class orchestration concept | Explicit pipeline parameter |
| **Partition mappings** | Built-in dependency reasoning | Manual plumbing or redesign |
| **Backfills** | First-class operational workflow | Series of triggered pipeline runs |

**Practical impact**: Passing `run_date` preserves only the label, not the whole partition engine.

### Scheduling and automation

| Aspect | Dagster | ZenML |
|---|---|---|
| **Scheduler ownership** | Dagster automation/schedules/sensors | Orchestrator-backed schedules and external triggers |
| **Data-aware policies** | Freshness + declarative automation | Must be rebuilt explicitly |
| **Event-driven triggers** | Sensors / asset sensors | External eventing or custom trigger service |

**Practical impact**: A plain cron schedule may preserve timing but not the original automation logic.

### Testing model

| Aspect | Dagster | ZenML |
|---|---|---|
| **Focus** | Asset/op/job definitions and Dagster execution helpers | Plain Python unit tests plus pipeline/step execution |
| **Graph object testing** | Common | Less central |
| **Migration implication** | Refactor tests toward Python function behavior and Dagster execution helpers | Refactor tests toward Python function behavior and artifact expectations |

---

## Migration Decision Tree

Use this to systematically classify a Dagster codebase before generating ZenML code.

```
1) Classify the source model
   ├── Mostly assets -> "asset_graph"
   ├── Mostly ops/graphs/jobs -> "op_job"
   └── Mixed -> inventory both before translating anything

2) Find the true run boundaries
   ├── Single tightly coupled job/subgraph -> candidate single ZenML pipeline
   ├── Asset groups with different schedules or selective materialization -> split into multiple pipelines
   └── Heavy unsupported semantics -> partial migration + redesign report

3) Classify compute nodes
   ├── @op -> translate directly to @step
   ├── @asset -> extract compute body into @step, preserve outputs as artifacts
   ├── @multi_asset -> multi-output step IF subset semantics not required
   ├── @graph_asset -> helper steps + terminal output artifact
   └── DynamicOutput -> dynamic pipeline or redesign

4) Classify storage and IO
   ├── Simple serialization IO manager -> materializer/artifact store mapping
   ├── IO manager with warehouse/file loading logic -> explicit source/sink steps
   └── mem-only or lazy-load semantics -> HIGH severity flag

5) Classify config and resources
   ├── Typed config -> pipeline params / YAML config
   ├── Credentials -> ZenML secrets or service connectors
   ├── Infra-wide clients -> stack components / orchestrator settings
   └── Per-step helper objects -> construct in code

6) Classify orchestration features
   ├── Simple schedule only -> Schedule(...) if target orchestrator supports it
   ├── Sensors / asset sensors -> FLAG HIGH, redesign to external trigger
   ├── Declarative automation / freshness -> FLAG HIGH, redesign to policy + trigger system
   ├── Partitions/backfills -> preserve partition key explicitly, flag non-trivial mappings
   └── Asset checks -> validation steps with metadata logging

7) Decide the migration result
   ├── No HIGH flags -> full migration with caveats documented
   ├── Some HIGH flags but compute mostly portable -> partial translation + TODOs + redesign guidance
   └── Unsupported semantics dominate -> stop after migration plan/report and explain why full auto-migration would be misleading
```

---

## ZenML Features with No Dagster Equivalent

These are capabilities the user gains after migration. Include relevant ones in the "What You Get for Free" section of the migration report.

| ZenML Feature | Value |
|---|---|
| **Artifact versioning and lineage** | Every step output is versioned and connected to the code and inputs that produced it. |
| **Step caching** | Steps can skip re-execution when code and inputs have not changed. |
| **Stack abstraction** | The same pipeline code can run on different stacks without rewriting the business logic. |
| **Service connectors** | Unified authentication abstraction across cloud providers and stack components. |
| **Pipeline deployments / external triggering patterns** | Strong patterns for API- or service-driven execution, especially when replacing sensors with external triggers. |
| **Model Control Plane** | First-class ML model tracking, promotion, and cross-pipeline lineage. |
| **Artifact-centric metadata and visualizations** | Useful when migrating data/ML workflows that need better run-to-artifact traceability. |
