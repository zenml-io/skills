# Gaps, Flags, and Behavioral Differences

This reference covers the Prefect patterns that are most dangerous to silently approximate during migration.

## Table of Contents

- [Must-Flag Patterns](#must-flag-patterns)
- [Behavioral Differences](#behavioral-differences)
- [Migration Decision Tree](#migration-decision-tree)
- [ZenML Features with No Prefect Equivalent](#zenml-features-with-no-prefect-equivalent)
- [Anti-Patterns in Migration](#anti-patterns-in-migration)

---

## Must-Flag Patterns

These patterns must **never** be silently approximated. Flag them in the migration report and require human review.

### Branching on runtime task outputs

**What Prefect does**: the flow can branch on a task result as ordinary Python because the flow body is executing imperatively.

**Why it matters**: a static ZenML pipeline cannot branch on artifact values while compiling the DAG.

**Migration options**:
- use `@pipeline(dynamic=True)` if the target stack supports it,
- redesign the branch to depend on pipeline parameters,
- or split the workflow into multiple pipelines.

### Runtime fan-out from `.submit()` / `.map()`

**What Prefect does**: task runners can create concurrent task executions based on runtime-discovered items.

**Why it matters**: ZenML dynamic pipelines can approximate this, but the execution semantics are different and support is narrower.

**Migration options**:
- use dynamic pipelines with `.map()` when same-run artifacts and orchestrator support line up,
- or redesign into multi-run orchestration / batch-inside-a-step.

**Important caveat**: dynamic pipelines are the closest escape hatch here, but they come with real limits. The current ZenML docs describe them as experimental, with limited orchestrator support and tighter execution-mode/failure-handling behavior than standard static pipelines. Surface those caveats before presenting a fan-out migration as "safe".

### `return_state=True`

**What Prefect does**: returns a terminal `State` instead of raising immediately.

**Why it matters**: ZenML does not use inspectable State objects as a first-class authoring primitive.

**Migration options**:
- redesign around explicit result envelopes,
- or move the inspection logic outside the pipeline.

### `allow_failure()`

**What Prefect does**: allows a downstream task to accept a failed upstream input.

**Why it matters**: ZenML has no dependency-level failure-tolerance feature with equivalent semantics.

**Migration rule**: do **not** replace this with a global continue-on-failure switch. That changes behavior.

### Manual State inspection and returned State objects

Flag any workflow that:
- inspects `.is_failed()` / `.is_completed()` or state types,
- returns `Failed(...)` / `Completed(...)` manually,
- branches on a `State` rather than on ordinary business data.

These patterns are dangerous because they turn Prefect's orchestration state into application control flow. ZenML does not expose a matching in-pipeline State object model.

### Pause / suspend

**What Prefect does**: can pause or suspend a flow run and later resume it.

**Why it matters**: ZenML has partial waiting/approval patterns, but not the same durable orchestration-state model across all scenarios.

**Migration options**:
- explicit approval/wait steps,
- external trigger-driven continuation,
- split the workflow into separate pipelines.

### Task-runner semantics

Flag any workflow where correctness or cost assumptions depend on:
- `ThreadPoolTaskRunner`
- `ProcessPoolTaskRunner`
- Dask / Ray task runners
- shared in-process concurrency behavior

These are not mere implementation details. They often shape the workflow's correctness, resource usage, or latency.

### Global concurrency and rate limiting

Flag any use of:
- `concurrency("name")`
- `rate_limit("name")`
- tag-based limits
- deployment/work-pool concurrency used for correctness

Prefect's slot-based concurrency/rate-limit model has no direct ZenML OSS equivalent.

### Caching with business semantics

Flag:
- `cache_key_fn`
- cache expiration/TTL relied on for correctness
- `refresh_cache` patterns that matter semantically

ZenML caching is real, but it is not "user-provided cache keys with the same meaning."

### Blocks, Deployments, and Automations used as a control plane

Flag:
- custom Blocks that mix secrets + runtime config + behavior,
- work pools that encode routing/business behavior,
- Automations that drive workflow logic,
- Cloud-only deployment behavior that the code depends on.

### Transactions and rollback hooks

Flag:
- transaction blocks,
- `on_commit`,
- `on_rollback`,
- any workflow where rollback is a first-class requirement.

This is redesign territory.

---

## Behavioral Differences

These differences are important even when migration is possible.

### Dynamic execution vs DAG compilation

| Aspect | Prefect | ZenML |
|---|---|---|
| Workflow shape | Can evolve while the flow is running | Static pipelines compile the DAG before step execution |
| Runtime branching | Native | Requires dynamic pipelines or redesign |
| Runtime fan-out | Native with task runners | Requires dynamic pipelines or redesign |

### State model

| Aspect | Prefect | ZenML |
|---|---|---|
| State object | First-class and inspectable | No equivalent in-pipeline object |
| Pause / suspend | Built into run state model | Partial wait/approval patterns only |
| Continue after failed upstream | `allow_failure()` | No dependency-level equivalent |

### Results vs artifacts

| Aspect | Prefect | ZenML |
|---|---|---|
| Output model | Results can be optionally persisted | Step outputs are versioned artifacts |
| Storage | Configurable result storage | Artifact store |
| Serialization | Result serializers | Materializers |
| Caching | Prefect task-state reuse | Artifact/code/input-based step caching |

### Concurrency

| Aspect | Prefect | ZenML |
|---|---|---|
| Primary parallelism model | Task runners inside a flow | Orchestrator/step execution model |
| Global concurrency / rate limits | Built-in named slot system | No direct OSS equivalent |
| Dask / Ray | First-class runner integrations | Usually embedded inside steps or re-architected |

### Configuration and deployment model

| Aspect | Prefect | ZenML |
|---|---|---|
| Typed config objects | Blocks | Split across secrets, connectors, stack config, YAML |
| Deployment abstraction | Deployment + work pool + worker | Orchestrator + schedule + stack + runtime config |
| Event-driven automation | Built-in Automations | External eventing or ZenML Pro-specific features |

---

## Migration Decision Tree

Use this to classify a Prefect workflow systematically:

```text
1) Classify the flow shape
   ├── Static, known before execution → prefer standard @pipeline
   └── Depends on task outputs / runtime fan-out → consider @pipeline(dynamic=True)

2) Check stateful control flow
   ├── Uses return_state=True / manual State inspection / returned State objects → FLAG HIGH
   ├── Uses allow_failure() → FLAG HIGH
   └── Uses pause/suspend → FLAG HIGH

3) Check concurrency semantics
   ├── Plain sequential / simple retry → translate normally
   ├── .submit() / .map() for throughput only → dynamic pipeline maybe OK
   ├── Dask/Ray or runner semantics critical → FLAG HIGH
   └── concurrency()/rate_limit() → FLAG HIGH

4) Check config/deployment
   ├── Secret-only Blocks → migrate to ZenML secrets
   ├── Infra/auth Blocks → service connectors / stack config
   ├── Deployments/work pools only for scheduling → approximate migration
   └── Automations/work pools/Cloud control plane drive business logic → FLAG HIGH

5) Decide
   ├── No HIGH flags → full migration with notes
   ├── Some HIGH flags but core path still useful → partial migration + redesign report
   └── Workflow built around state/control-plane features → architectural redesign
```

---

## ZenML Features with No Prefect Equivalent

These are worth mentioning in the migration report as "what you get for free".

| Feature | Value |
|---|---|
| **Artifact versioning and lineage** | Every step output is tracked as a first-class artifact. |
| **Stack abstraction** | The same pipeline code can move across infrastructure by switching stacks. |
| **Service connectors** | Standardized cloud auth and credential management. |
| **Model Control Plane** | First-class model tracking and promotion workflows. |
| **Pipeline deployments (HTTP services)** | Long-running pipeline-serving mode — a different capability, not a Prefect Deployment clone. |
| **Strong artifact-centric reproducibility** | Inputs, outputs, and lineage are explicit and inspectable. |

---

## Anti-Patterns in Migration

| Anti-pattern | Why it's wrong | What to do instead |
|---|---|---|
| Replace `allow_failure()` with a global continue-on-failure setting | Changes dependency semantics | Return explicit success/error artifacts |
| Convert runtime branching to static `if` on step outputs | Static pipelines cannot do that safely | Use dynamic pipelines or redesign |
| Assume `.submit()` / `.map()` preserve Prefect runner behavior | Parallelism model differs | Re-check correctness and infra assumptions |
| Convert all Blocks into env vars | Loses schema and concern separation | Split into secrets, connectors, stack config, YAML |
| Treat Prefect Deployment as ZenML pipeline deployment | They solve different problems | Map to schedules/orchestrators unless HTTP serving is truly needed |
| Silently drop concurrency/rate limits | Can change correctness and overload external systems | Add explicit throttling or platform-level controls |
| Ignore `cache_key_fn` | Can change business semantics | Flag and redesign cache behavior explicitly |
