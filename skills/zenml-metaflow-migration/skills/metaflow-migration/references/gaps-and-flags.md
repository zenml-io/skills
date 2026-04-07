# Gaps, Flags, and Behavioral Differences

This reference covers the Metaflow patterns that are most dangerous to silently approximate during migration -- the spots where the rewritten pipeline may still run while meaning something different.

## Table of Contents

- [Must-Flag Patterns](#must-flag-patterns)
- [Behavioral Differences](#behavioral-differences)
- [Migration Decision Tree](#migration-decision-tree)
- [ZenML Features with No Metaflow Equivalent](#zenml-features-with-no-metaflow-equivalent)

---

## Must-Flag Patterns

These patterns must **never** be silently approximated. Flag them in the migration report and require human review.

### `@catch`

**What Metaflow does**: converts a failing step into a successful continuation point and can attach exception data to an artifact.

**Why it matters**: this changes the pipeline's failure story. Downstream code may expect the flow to continue even when a step blew up.

**ZenML gap**: no direct placeholder-success equivalent.

**Redesign approaches**:
- return an explicit result/error envelope from the step
- split recovery into a separate pipeline
- move notification behavior into hooks

### `merge_artifacts(inputs)`

**What Metaflow does**: resolves join-time ambiguity by pulling forward branch artifacts implicitly.

**Why it matters**: naive rewrites tend to keep only one branch or guess a winner.

**ZenML gap**: no implicit merge primitive.

**Redesign approaches**:
- write an explicit reducer/join step
- list each incoming branch artifact intentionally
- implement conflict resolution in code

### `foreach` with runtime-shaped semantics

**What Metaflow does**: dynamically fans out one task per item and preserves `self.input`, task identity, and join-time aggregation semantics.

**Why it matters**: rewriting it as a plain Python loop loses orchestration semantics and often parallelism.

**ZenML gap**: dynamic pipelines can express similar intent, but the semantics and support surface differ.

**Redesign approaches**:
- use `@pipeline(dynamic=True)` plus `.map()` when supported
- the current docs list support on `local`, `local_docker`, `kubernetes`, `sagemaker`, `vertex`, and `azureml`
- for manual loops, use `.load()` for control-flow decisions and `.chunk(idx)` for DAG wiring
- if support is uncertain, flag the fan-out for redesign instead of pretending it is equivalent

### Conditional branching based on runtime artifacts

**What Metaflow does**: can route the graph based on values produced during the run.

**Why it matters**: static ZenML pipelines cannot do this directly.

**ZenML gap**: requires dynamic pipelines or redesign.

**Redesign approaches**:
- use dynamic pipelines only when the target stack supports them
- otherwise refactor into separate pipelines or explicit orchestration logic

### Recursion

**What Metaflow does**: newer versions support recursive control-flow patterns.

**Why it matters**: recursion changes graph shape and termination behavior.

**ZenML gap**: no documented general recursion primitive.

**Redesign approaches**:
- move the loop outside the pipeline
- run repeated pipelines from a controller process
- redesign into iterative dynamic orchestration if truly appropriate

### `resume` and `@checkpoint`

**What Metaflow does**: lets users restart or recover work using run/task identity and checkpoint state.

**Why it matters**: teams often depend on this for long-running training or brittle infrastructure.

**ZenML gap**: caching and artifact reuse help, but they are not semantic equivalents.

**Redesign approaches**:
- model checkpoint state as explicit artifacts
- persist intermediate outputs to durable storage
- document the changed recovery behavior clearly

### `@batch`

**What Metaflow does**: sends work to AWS Batch with Batch-shaped compute semantics.

**Why it matters**: compute placement, retry behavior, and runtime assumptions may be part of correctness or cost control.

**ZenML gap**: no portable direct `@batch` equivalent.

**Redesign approaches**:
- choose the target runtime explicitly (Kubernetes, SageMaker, Databricks, etc.)
- express resources and environment in stack/runtime settings

### Portable `@timeout`

**What Metaflow does**: applies timeout semantics at the decorator level.

**Why it matters**: timeouts interact with retries and failure continuation.

**ZenML gap**: timeout support is backend-specific, not a universal core primitive.

**Redesign approaches**:
- use backend-specific job/pod timeout settings
- implement application-level timeout handling where needed

### `@trigger` / `@trigger_on_finish`

**What Metaflow does**: ties event-driven or flow-chaining behavior to the deployment model.

**Why it matters**: the workflow may depend on event-driven orchestration, not just scheduled batch runs.

**ZenML gap**: no direct decorator-level equivalent.

**Redesign approaches**:
- external eventing + pipeline deployment
- API-triggered runs
- CI/CD or message-bus orchestration

### Business logic that depends on `current.*`

**What Metaflow does**: exposes a broad runtime context surface.

**Why it matters**: code may depend on metadata that does not exist in ZenML the same way.

**ZenML gap**: step context is narrower.

**Redesign approaches**:
- replace runtime introspection with explicit parameters or metadata logging
- compute needed metadata inside the pipeline contract

### Outerbounds-only features

**What Metaflow / Outerbounds does**: adds features like Fast Bakery, `@gpu_profile`, deployment endpoints, and assets APIs.

**Why it matters**: users may think these are "just Metaflow".

**ZenML gap**: the overlapping capabilities exist, but often under different abstractions.

**Redesign approaches**:
- classify each feature by intent
- translate to stack/deployment/model concepts only when the behavior is genuinely similar
- otherwise flag it as redesign work

---

## Behavioral Differences

### Data passing: `self.*` vs explicit artifacts

| Aspect | Metaflow | ZenML |
|---|---|---|
| Artifact creation | implicit via `self.<name> = ...` | explicit via step returns |
| Downstream access | instance attributes | function inputs |
| Serialization model | pickle-like artifact persistence | materializer-driven typed artifacts |
| Hidden dependency risk | high | lower, because dependencies must be named |

### Execution model

| Aspect | Metaflow | ZenML |
|---|---|---|
| Workflow structure | `FlowSpec` + `self.next(...)` | `@pipeline` + step calls |
| Local execution | per-step process isolation | usually same active environment unless remote runtime is used |
| Dynamic graph shape | built into Metaflow model | opt-in via dynamic pipelines |
| Join semantics | special join step behavior | explicit step contracts |

### Environment management

| Aspect | Metaflow | ZenML |
|---|---|---|
| Dependency story | decorators like `@conda`, `@pypi`, Fast Bakery | Docker/image settings and stack runtime |
| Backend selection | decorators and CLI overlays | active stack + runtime settings |
| Secret injection | `@secrets` | secrets store + service connectors |

### Compute management

| Aspect | Metaflow | ZenML |
|---|---|---|
| Resource expression | decorators such as `@resources`, `@batch`, `@kubernetes` | `ResourceSettings`, stack components, step operators |
| Backend identity | often explicit in source code/decorators | often chosen by stack configuration |
| Portability | depends on Metaflow runtime | depends on the target ZenML stack |

### Artifact lifecycle and lineage

| Aspect | Metaflow | ZenML |
|---|---|---|
| Artifact naming | often derived from `self.*` | derived from step outputs and type information |
| Lineage object model | Run / Step / Task / Artifact | Pipeline / Run / StepRun / ArtifactVersion / Model |
| Visualization | cards and client inspection | dashboard, metadata, visualizations, models |

### Resume vs caching

| Aspect | Metaflow | ZenML |
|---|---|---|
| Reuse mechanism | resume/checkpoint by prior run/task state | cache/artifact reuse by code, inputs, settings, lineage |
| Failure recovery story | explicit operational feature | partial approximation via caching and artifact design |
| Safe to claim "equivalent"? | no | no |

---

## Migration Decision Tree

Use this text-first decision procedure when classifying a Metaflow pattern:

```text
INPUT: flow pattern, decorator set, artifact usage, control-flow shape

1. Is the flow linear and artifact dependencies are obvious?
   - Yes -> translate directly to static ZenML pipeline
   - No -> continue

2. Does the flow branch and later join?
   - Yes -> design an explicit join step
   - If join relied on merge_artifacts(inputs) or broad implicit state -> FLAG HIGH

3. Does the flow use foreach, runtime branching, or recursion?
   - If yes and target stack supports dynamic pipelines -> translate approximately and note limitations
   - If yes and support is uncertain -> FLAG HIGH and propose redesign

4. Does the flow use @catch, @checkpoint, resume-heavy recovery, or @trigger*?
   - Yes -> FLAG HIGH; do not claim equivalence

5. Does the flow use platform decorators (@batch, @kubernetes, @conda, @pypi, Fast Bakery)?
   - Translate by intent:
     - dependencies -> DockerSettings
     - resources -> ResourceSettings
     - compute backend -> stack/runtime selection
   - If the original behavior depended on a specific provider contract -> FLAG MEDIUM/HIGH

6. Does the code depend on current.*, metaflow.client, Runner, or Deployer?
   - If only metadata lookup -> approximate via ZenML Client or step context
   - If control flow depends on these APIs -> FLAG for design review

7. After classification:
   - Direct only -> full auto-migration
   - Some approximate, no high-severity blockers -> migrate with caveats
   - Any high-severity blockers -> partial migration + migration report + redesign notes
   - Two or more high-severity blockers -> also generate a community-help / Slack-ready message
```

---

## ZenML Features with No Metaflow Equivalent

Migration is not only about losses. ZenML also gives users capabilities that often need to be surfaced explicitly after the rewrite.

| ZenML Feature | Why it matters after migration |
|---|---|
| Typed artifacts + materializers | Makes data contracts explicit and inspectable |
| Artifact lineage | Easier to trace where data or models came from |
| Step caching | Saves reruns when code and inputs have not changed |
| Stack abstraction | Same pipeline code can move across local and cloud stacks |
| Service connectors | Centralized auth for cloud resources |
| Model Control Plane | Rich model/version organization beyond raw run artifacts |
| Pipeline deployments | Turn pipelines into HTTP-triggered services |
| Snapshots | Re-run immutable pipeline snapshots from the SDK, CLI, dashboard, or API |

Use this section in the migration report as the "what you get for free" bridge: it helps the user understand not just what changed, but what became possible.
