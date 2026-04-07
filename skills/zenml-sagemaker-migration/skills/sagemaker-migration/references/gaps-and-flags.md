# Gaps, Flags, and Behavioral Differences

This reference covers the patterns that are most dangerous to silently approximate during a SageMaker-to-ZenML migration -- the places where the two systems behave fundamentally differently, and where a naive translation changes the pipeline's actual behavior.

## Table of Contents

- [Must-Flag Patterns](#must-flag-patterns)
- [Behavioral Differences](#behavioral-differences)
- [Migration Decision Tree](#migration-decision-tree)
- [ZenML Features with No SageMaker Equivalent](#zenml-features-with-no-sagemaker-equivalent)
- [Anti-Patterns in Migration](#anti-patterns-in-migration)

---

## Must-Flag Patterns

These patterns must **never** be silently approximated. Flag them in the migration report and require human review.

### `PropertyFile` + `JsonGet` + step `.properties`

**What SageMaker does:** it treats step outputs as backend-visible placeholder structures. Downstream nodes can read specific JSON fields or step properties without materializing the whole output as a normal Python object.

**Why it matters:** this is one of the deepest semantic differences in the migration. SageMaker's placeholder system is part of the orchestration layer. ZenML uses artifacts and Python values instead.

**ZenML gap:** there is no equivalent placeholder-expression system.

**Redesign approach:** return typed artifacts from steps, then read those artifacts normally in downstream steps or in a dynamic pipeline for small control values.

### `ConditionStep` over runtime placeholder values

**What SageMaker does:** `ConditionStep` evaluates conditions on backend placeholder values and controls which downstream steps become ready.

**Why it matters:** backend condition evaluation is not the same as Python control flow. Caching behavior, evaluation timing, and artifact materialization all change.

**Redesign approach:**
- If the branch depends only on pipeline parameters, use normal Python control flow in the pipeline definition.
- If it depends on step outputs, use `@pipeline(dynamic=True)` and `.load()` only for small control artifacts.
- If the pipeline also needs native SageMaker-backed scheduling, flag the design immediately because dynamic pipelines cannot currently be scheduled on the SageMaker orchestrator.

### `TuningStep`

**What SageMaker does:** it launches a managed hyperparameter-tuning job that can spawn many training jobs.

**Why it matters:** collapsing that to one training step destroys the original behavior.

**Redesign approach:** keep HPO explicit -- either launch SageMaker HPO from a ZenML step or deliberately redesign to another HPO system.

### `TransformStep`

**What SageMaker does:** it wraps SageMaker Batch Transform as a first-class pipeline step.

**Why it matters:** ZenML has no first-class batch-transform step. A naive rewrite can silently remove AWS-managed inference semantics.

**Redesign approach:** keep Batch Transform explicit inside a ZenML step, or intentionally redesign to ordinary offline inference code.

### `ModelStep`, `CreateModelStep`, and `RegisterModel`

**What SageMaker does:** it creates SageMaker model resources and/or registers model packages in SageMaker Model Registry.

**Why it matters:** ZenML Model Control Plane is **not** a drop-in replacement for SageMaker Model Registry.

**Redesign approach:** choose one of these deliberately:
- keep SageMaker registration as explicit AWS-native step logic
- redesign governance around ZenML MCP
- do both, but make the dual-registration strategy explicit

### `CallbackStep`

**What SageMaker does:** it uses callback-token semantics and an external completion protocol.

**Why it matters:** there is no native ZenML equivalent.

**Redesign approach:** keep this portion as explicit AWS workflow logic, or move the orchestration boundary to another system such as Step Functions.

### `LambdaStep`

**What SageMaker does:** it invokes Lambda with Lambda-specific I/O and timeout semantics.

**Why it matters:** a plain Python step is not the same thing.

**Redesign approach:** invoke Lambda explicitly from a step if you truly need Lambda, and document the resulting AWS dependency.

### SageMaker Feature Store parity claims

**What SageMaker does:** it provides AWS-native feature-store workflows and APIs.

**Why it matters:** do **not** tell the user that SageMaker Feature Store is meaningfully covered by the ZenML feature-store abstraction. That would overstate current support and send the migration in the wrong direction.

**Redesign approach:** keep SageMaker Feature Store usage explicit inside AWS-native steps, or treat it as custom integration work. If the user wants first-class support, point them to `zenml.io/slack` to discuss it with the community/team, or suggest contributing the integration themselves.

### Scheduling lifecycle on the SageMaker orchestrator

**What SageMaker / ZenML do:** ZenML can create cron / interval / one-time schedules on the SageMaker orchestrator, but native schedule management is not supported there, and dynamic pipelines cannot currently be scheduled.

**Why it matters:** users often assume "scheduled" also means "updateable in place" and "works for any pipeline shape." That is false here.

**Redesign approach:** document the lifecycle clearly. If the original pipeline combined branching and native SageMaker scheduling, flag it and discuss alternatives.

### Warm pools, training-step path, and output-mode constraints

**What ZenML documents:** using multichannel output or output modes other than `EndOfJob` prevents using the training-step path and warm pools on the SageMaker orchestrator.

**Why it matters:** this is the kind of runtime detail that silently changes startup behavior and cost if you ignore it.

**Redesign approach:** surface the trade-off in the migration report and decide whether preserving output semantics or preserving warm-pool / training-step behavior matters more.

---

## Behavioral Differences

### Step-type DSL vs uniform steps

| Aspect | SageMaker Pipelines | ZenML |
|---|---|---|
| **Primary unit** | Step class (`ProcessingStep`, `TrainingStep`, `TransformStep`, etc.) | Python `@step` |
| **Where service semantics live** | In the graph DSL | Often inside step code or orchestrator settings |
| **How the graph is authored** | Workflow objects and placeholders | Python function composition |

### Placeholders vs artifacts

| Aspect | SageMaker | ZenML |
|---|---|---|
| **Data handoff** | Step properties, `PropertyFile`, `JsonGet`, parameters | Typed artifacts and Python values |
| **Evaluation model** | Backend placeholder evaluation | Python evaluation and artifact materialization |
| **Typical control-flow pattern** | Conditions over placeholder values | Dynamic pipelines with `.load()` for small control artifacts |
| **Serialization model** | JSON files / service-specific outputs | Materializers and artifact store persistence |

### Scheduling lifecycle

| Aspect | Native SageMaker | ZenML on SageMaker |
|---|---|---|
| **Schedule types** | Cron, rate, at | Cron, interval, one-time |
| **Schedule updates** | Managed via SageMaker APIs | Not natively managed by ZenML on SageMaker |
| **Dynamic pipeline scheduling** | Not the same concern in native SageMaker DSL | Dynamic pipelines work on the SageMaker orchestrator, but scheduling them is not supported |

### Model governance and experiments

| Aspect | SageMaker | ZenML |
|---|---|---|
| **Model registry** | SageMaker Model Registry / model packages | ZenML MCP / models |
| **Experiments** | SageMaker Experiments | Experiment trackers + ZenML runs |
| **Feature store** | SageMaker Feature Store | No practical built-in SageMaker-equivalent wrapper; explicit AWS usage or custom integration |
| **Main migration risk** | Assuming the names imply the same lifecycle semantics | They do not |

---

## Migration Decision Tree

Text-based decision procedure for translating SageMaker patterns to ZenML:

```text
INPUT: construct_type, data_flow_pattern, scheduling_requirements, service_coupling

IF construct_type IN {ProcessingStep, TrainingStep}:
  EMIT normal @step
  IF user wants to keep SageMaker runtime details:
    MOVE instance/S3/tag/runtime knobs into SagemakerOrchestratorSettings

ELSE IF construct_type == ConditionStep:
  IF branch depends only on pipeline parameters:
    EMIT normal Python conditional in the pipeline
  ELSE:
    EMIT @pipeline(dynamic=True)
    USE .load() only for small control artifacts
    IF scheduling_requirements include SageMaker-native scheduling:
      FLAG blocker DYNAMIC_PIPELINE_SCHEDULING

ELSE IF construct_type IN {PropertyFile, JsonGet, step.properties}:
  REPLACE placeholder access with typed artifacts
  FLAG warn DATA_MODEL_REWRITE

ELSE IF construct_type == TuningStep:
  EMIT explicit HPO step design
  FLAG warn HPO_NOT_A_NORMAL_STEP

ELSE IF construct_type == TransformStep:
  EMIT explicit Batch Transform step
  FLAG warn AWS_NATIVE_INFERENCE

ELSE IF construct_type IN {ModelStep, CreateModelStep, RegisterModel}:
  ASK which system should be the source of truth:
    - SageMaker registry
    - ZenML MCP
    - both
  EMIT explicit strategy

ELSE IF construct_type IN {CallbackStep, LambdaStep}:
  FLAG blocker EXPLICIT_AWS_WORKFLOW_REQUIRED
  RECOMMEND keeping AWS-native logic explicit or redesigning the orchestration boundary

ELSE IF construct_type IN {QualityCheckStep, ClarifyCheckStep, EMRStep, NotebookJobStep, AutoMLStep}:
  EMIT explicit service-launching @step
  FLAG warn SERVICE_SPECIFIC_SEMANTICS

IF data_flow_pattern uses raw S3 URIs only to mimic placeholders:
  PREFER artifacts instead
ELSE IF AWS-native service calls require URIs:
  KEEP URIs explicit and document the non-portable dependency
```

---

## ZenML Features with No SageMaker Equivalent

These are not migration problems. They are capabilities the user gets **after** migration:

- **Artifact lineage and versioning** for every persisted output
- **Step caching** based on code + inputs + parameters
- **Stack abstraction** so the same pipeline can run on other backends later
- **Service connectors** for managed cloud authentication
- **Model Control Plane** as an additive governance layer

Call these out in the "What You Get for Free After Migration" section of the migration report.

---

## Anti-Patterns in Migration

| Anti-pattern | Why it's wrong | What to do instead |
|---|---|---|
| Keeping `PropertyFile` / `JsonGet` wrappers everywhere | Preserves the old DSL's complexity and hides ZenML's artifact model | Return typed artifacts and access them normally |
| Treating runtime knobs as normal business parameters | Mixes infrastructure decisions with pipeline semantics | Move them into `SagemakerOrchestratorSettings` |
| Silently translating `ConditionStep` to Python without explanation | Changes where and when the decision is made | Flag the semantic difference explicitly |
| Translating `TuningStep` to one training run | Destroys the original optimization behavior | Keep HPO explicit |
| Saying "Model Registry == MCP" | Misleads the user about governance semantics | Explain the difference and choose intentionally |
| Saying "SageMaker Feature Store == ZenML feature store" | Overstates current support and leads to a bad migration plan | Keep SageMaker Feature Store explicit or treat it as custom integration work |
| Claiming Lambda / callback semantics are preserved | They are not native ZenML concepts | Keep them as explicit AWS integrations or redesign |
| Keeping raw S3 URIs for ordinary inter-step data flow | Throws away artifact lineage and caching benefits | Use artifacts unless AWS-native APIs require URIs |
