# Gaps, Flags, and Behavioral Differences

This reference covers the patterns that are most dangerous to silently approximate during a Flyte → ZenML migration — the places where similar-looking code would tell a different execution story after translation.

## Table of Contents

- [Must-Flag Patterns](#must-flag-patterns)
- [Behavioral Differences](#behavioral-differences)
- [Migration Decision Tree](#migration-decision-tree)
- [ZenML Features with No Flyte Equivalent](#zenml-features-with-no-flyte-equivalent)

---

## Must-Flag Patterns

These patterns must **never** be silently approximated. Flag them in the migration report and require human review.

### FlyteFile / FlyteDirectory pointer semantics

**What Flyte does:** `FlyteFile` and `FlyteDirectory` are remote-aware transport types. Flyte can localize them into the execution environment automatically.

**Why it matters:** a plain string or local path in ZenML may work in local testing and then quietly break on remote execution.

**ZenML gap:** `Path` is the closest artifact type, but it does not automatically preserve remote-reference semantics or provenance.

**Redesign approaches:**
- use `Path` plus explicit metadata for the source URI
- build a wrapper type + custom materializer
- use `ExternalArtifact` / `register_artifact` when the file comes from outside the current run

### StructuredDataset / FlyteSchema metadata contracts

**What Flyte does:** these types represent more than “some table.” They can carry schema, reader/writer, or backend-format expectations.

**Why it matters:** translating them to `Any` or a bare dataframe without validation hides assumptions that were previously encoded in the Flyte transport layer.

**ZenML gap:** dataframe artifacts are a good payload match, but not a full metadata-contract match.

**Redesign approaches:**
- use concrete dataframe/table types
- add a validator step
- log the missing format/schema metadata explicitly
- create a custom materializer if the metadata is operationally important

### `map_task()` advanced semantics

**What Flyte does:** `map_task()` is a first-class fan-out primitive with engine-level behavior. Flyte can also express concurrency and partial-success semantics.

**Why it matters:** a naive `.map()` or plain Python loop can silently change parallelism, retry behavior, and failure semantics.

**ZenML gap:** dynamic `.map()` is only an approximate match. There is no portable equivalent for `min_success_ratio`, and concurrency behavior is backend-specific.

**Redesign approaches:**
- use `@pipeline(dynamic=True)` + `.map()` when the target orchestrator supports it
- batch or chunk inputs if backend concurrency must be controlled
- redesign partial-success behavior as an explicit reducer / threshold step

### `conditional()` and runtime graph shaping

**What Flyte does:** models runtime branching inside the workflow engine using promises.

**Why it matters:** the graph that actually runs can differ based on runtime data.

**ZenML gap:** ZenML dynamic pipelines can branch with `.load()`, but the control flow is Python-side and not the same engine primitive.

**Redesign approaches:**
- use a dynamic pipeline with `.load()` when the branch logic is simple and backend support exists
- move the decision inside a step and keep the outer pipeline simpler
- split pre-branch and post-branch logic into separate pipelines

### `@eager`

**What Flyte does:** supports async imperative orchestration using Flyte tasks.

**Why it matters:** eager execution is closer to a warm interactive runner than to a static or dynamic declarative graph.

**ZenML gap:** no direct core equivalent.

**Redesign approaches:**
- redesign as a normal static pipeline
- redesign as a dynamic pipeline
- move the imperative coordination outside ZenML and let ZenML own the durable steps

### LaunchPlans beyond simple scheduling

**What Flyte does:** `LaunchPlan` is the registered execution surface with schedules, defaults, fixed inputs, and notifications.

**Why it matters:** if you migrate only the cron expression, the user loses part of the original behavior without realizing it.

**ZenML gap:** `Schedule` only covers the scheduling slice.

**Redesign approaches:**
- map schedule to `Schedule(...)`
- map defaults to pipeline defaults or config
- map fixed inputs to wrapper pipelines or deployment-specific config
- map notifications to hooks / alerters / external alerting

### `interruptible=True` and timeouts

**What Flyte does:** exposes task-level spot / preemptible intent and task timeout semantics.

**Why it matters:** cost profile, retry story, and “hung task” behavior can change materially.

**ZenML gap:** no portable core equivalent for interruptible semantics, and timeout handling is backend-specific.

**Redesign approaches:**
- move spot/preemptible behavior into the target orchestrator configuration
- keep retries explicit with `StepRetryConfig`
- split very long steps into smaller idempotent units

### ContainerTask and plugin `task_config`

**What Flyte does:** `ContainerTask` and many plugin-backed tasks represent execution contracts, not just ordinary Python callables.

**Why it matters:** if you translate only the Python wrapper, you lose the actual runtime boundary and IO protocol.

**ZenML gap:** no first-class raw container task primitive, and no 1:1 translation for plugin config objects.

**Redesign approaches:**
- wrap the external job as a submission step
- use a step operator when ZenML has one for the target system
- preserve explicit input/output staging

### Reference entities

**What Flyte does:** `reference_task`, `reference_workflow`, and `reference_launch_plan` let a workflow point to already-registered entities in other projects.

**Why it matters:** that is an architecture boundary, not just a code import.

**ZenML gap:** no direct equivalent for cross-project registered entity references.

**Redesign approaches:**
- shared libraries
- API/service boundaries
- explicit artifact exchange or external job triggers

### Intra-task checkpointing

**What Flyte does:** allows a task to save progress and resume partway through.

**Why it matters:** collapsing checkpointed logic into one large ZenML step makes retries more expensive and brittle.

**ZenML gap:** no core equivalent.

**Redesign approaches:**
- split the task into smaller idempotent steps
- persist intermediate outputs as artifacts
- redesign the task as chunked processing

### Union-only extensions

**What Flyte / Union does:** adds higher-level hosted and commercial features such as Actors, reusable containers, and other platform-specific semantics.

**Why it matters:** these can look like “just another option flag” when they are really architecture-level features.

**ZenML gap:** some concepts have intent-level approximations, but several do not.

**Redesign approaches:**
- treat Actors as a redesign boundary
- treat reusable containers as backend optimization, not portable code behavior
- keep Union Artifacts / Channels conservative if the semantics are unclear

---

## Behavioral Differences

These are the semantic differences that are often safe to translate, but important to explain clearly to the user.

### Type system

| Aspect | Flyte | ZenML |
|---|---|---|
| cross-boundary type story | richer transport types and runtime localization | Python types + materializers |
| file / directory types | `FlyteFile`, `FlyteDirectory` | usually `Path` artifacts |
| tabular types | `StructuredDataset`, `FlyteSchema` | dataframe / table artifacts |
| complex models | dataclass / Pydantic support in workflow transport | explicit serializer/materializer strategy |
| arbitrary objects | extensible type adaptation | temporary cloudpickle escape hatch, but not a stable endpoint |

**Practical impact:** migrate business payloads directly, but migrate transport semantics deliberately.

### Containerization

| Aspect | Flyte | ZenML |
|---|---|---|
| image customization | task image via `ImageSpec` or task image fields | step/pipeline image settings via `DockerSettings` |
| build lifecycle | part of Flyte packaging story | tied to the ZenML stack / image-build flow |
| raw non-Python execution | `ContainerTask` | no core equivalent |

### Parallel execution

| Aspect | Flyte | ZenML |
|---|---|---|
| fan-out primitive | `map_task()` | dynamic `.map()` |
| graph shaping | engine-level | Python-side dynamic execution |
| concurrency control | explicit `concurrency` | backend-specific |
| partial success | `min_success_ratio` | no portable equivalent |

### Caching

| Aspect | Flyte | ZenML |
|---|---|---|
| enable / disable | task-level cache flag | step-level cache flag |
| explicit version knob | `cache_version` | no verified identical core field in current repo guidance |
| hashing / serialization | tied to Flyte task interface behavior | influenced by materializers and step behavior |

### Resource management / spot handling

| Aspect | Flyte | ZenML |
|---|---|---|
| CPU / memory / GPU | task `Resources` plus requests / limits | `ResourceSettings` |
| interruptible / spot | first-class `interruptible=True` | backend-specific config |
| timeouts | first-class task timeout | backend-specific timeout handling |

### Scheduling / LaunchPlans

| Aspect | Flyte | ZenML |
|---|---|---|
| schedule | LaunchPlan schedule | `Schedule` |
| execution surface | LaunchPlan is the registered trigger object | pipeline + deployment/config tooling |
| frozen inputs | `fixed_inputs` | wrapper pipeline / deployment config |
| defaults | `default_inputs` | pipeline defaults / YAML config |
| notifications | LaunchPlan notifications | hooks / alerters / external alerts |

---

## Migration Decision Tree

Use this procedure on every Flyte component:

```text
1) Start with the authoring shape
   ├── Plain @task / @workflow with only primitives and collections
   │   -> translate first to @step / @pipeline
   ├── @dynamic
   │   -> candidate for @pipeline(dynamic=True)
   └── @eager
       -> FLAG HIGH: redesign boundary

2) Inspect all inputs and outputs
   ├── FlyteFile / FlyteDirectory
   │   -> choose Path vs wrapper type + materializer intentionally
   ├── StructuredDataset / FlyteSchema
   │   -> choose concrete dataframe/table + validation/materializer strategy
   ├── metadata-bearing Annotated / complex transport types
   │   -> FLAG for explicit metadata redesign
   └── plain Python types
       -> direct / near-direct candidate

3) Ask whether graph shape depends on runtime data
   ├── No
   │   -> static pipeline likely works
   ├── Yes, via @dynamic / conditional / map_task
   │   -> dynamic pipeline candidate
   └── Yes, but depends on engine-only semantics
       -> FLAG HIGH / redesign

4) Check execution metadata
   ├── retries
   │   -> StepRetryConfig
   ├── resources
   │   -> ResourceSettings
   ├── images
   │   -> DockerSettings
   ├── timeout / interruptible
   │   -> FLAG as backend-specific
   └── cache_version
       -> explicit cache-buster config

5) Check scheduling and trigger surface
   ├── LaunchPlan schedule only
   │   -> Schedule(...)
   ├── LaunchPlan + defaults/fixed inputs/notifications
   │   -> split into explicit config + wrapper/deployment concepts
   └── reference_launch_plan
       -> FLAG HIGH: redesign

6) Check external execution boundaries
   ├── plugin task / external job family
   │   -> intent-level mapping (step operator or submission step)
   ├── ContainerTask
   │   -> FLAG HIGH: redesign
   ├── reference_task / reference_workflow
   │   -> FLAG HIGH: architecture boundary
   └── checkpointing / Union Actors
       -> FLAG HIGH: redesign

7) Classify final result
   ├── Semantics mostly preserved
   │   -> direct
   ├── Same intent, different runtime story
   │   -> approximate
   └── No safe equivalent
       -> absent
```

---

## ZenML Features with No Flyte Equivalent

These are capabilities worth mentioning in the migration report as "what you get for free" after migration.

| ZenML Capability | Flyte Equivalent | Notes |
|---|---|---|
| `register_artifact` / `ExternalArtifact` as explicit lineage tools | no close Flyte core equivalent | Useful for bringing pre-existing data into lineage cleanly |
| Model Control Plane with model versioning and promotion stages | no Flyte core equivalent | Strong gain for ML lifecycle management |
| service connectors as a first-class auth abstraction | no close Flyte core equivalent | Centralized cloud auth story is stronger here |
| YAML-driven multi-environment pipeline configuration | no close Flyte core equivalent | Helpful when replacing LaunchPlan variants with environment-specific config |
| step success / failure hooks as core callback surfaces | only loose approximation via LaunchPlan notifications | More callback-like than Flyte's notification object model |
