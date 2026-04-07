# Gaps, Flags, and Behavioral Differences

This reference covers the AzureML patterns that are most dangerous to silently approximate during migration -- the places where AzureML SDK v2 and ZenML behave differently enough that a naive translation changes the workflow's real behavior.

## Table of Contents

- [Must-Flag Patterns](#must-flag-patterns)
- [Behavioral Differences](#behavioral-differences)
- [Migration Decision Tree](#migration-decision-tree)
- [ZenML Features with No AzureML Equivalent](#zenml-features-with-no-azureml-equivalent)
- [Anti-Patterns in Migration](#anti-patterns-in-migration)

---

## Must-Flag Patterns

These patterns must **never** be silently approximated. Flag them in the migration report and require human review.

### AzureML environments with complex build and dependency semantics

**What AzureML does**: `Environment(...)` objects can be reusable, registered assets with image + Conda semantics, curated environments, and Azure-managed build lifecycles.

**Why it matters**: Environment identity may be part of governance, reuse, or reproducibility -- not just a Docker detail.

**ZenML gap**: ZenML uses `DockerSettings`, not a first-class environment asset registry.

**Recommended path**:
- Translate the operational runtime into `DockerSettings`
- Explicitly document the loss of Azure environment asset semantics
- Keep Azure environment assets external if other teams still depend on them

### Sweep jobs (HPO)

**What AzureML does**: AzureML sweep jobs are first-class hyperparameter optimization primitives with Azure-native scheduling, metrics, and policy behavior.

**Why it matters**: Replacing a sweep with a plain loop changes execution semantics, observability, and failure behavior.

**ZenML gap**: No native sweep-job primitive.

**Recommended path**:
- Keep Azure sweep native and call it from a step, or
- Redesign explicitly around Optuna or another HPO library

### Parallel jobs for batch scoring

**What AzureML does**: Azure parallel jobs are designed for large-scale mini-batch scoring and operational parallel inference.

**Why it matters**: They are not just a generic for-each loop.

**ZenML gap**: Dynamic pipelines and `.map()` can resemble fan-out, but they are not a safe semantic replacement.

**Recommended path**:
- Keep the Azure parallel job native, or
- Redesign intentionally after validating partitioning, concurrency, retries, and output semantics

### YAML-defined and registered components

**What AzureML does**: YAML components and registered components act as reusable Azure assets with versions and potentially cross-team dependencies.

**Why it matters**: The registry identity may matter as much as the code body.

**ZenML gap**: ZenML has no component registry primitive.

**Recommended path**:
- Rewrite YAML-defined behavior as Python `@step` code
- Keep Azure registered components external if downstream consumers still depend on them

### MLTable and Azure data assets

**What AzureML does**: MLTable and asset IDs can carry reproducibility, schema, storage, and governance meaning.

**Why it matters**: Treating them as "just a pandas DataFrame" can erase an important contract.

**ZenML gap**: No first-class MLTable or Azure data asset type.

**Recommended path**:
- Keep Azure asset references explicit when identity matters
- Only collapse into ZenML artifacts when the asset identity is not part of the workflow contract

### Managed online endpoints and batch endpoints

**What AzureML does**: AzureML provides managed deployment primitives for serving and endpoint-based batch inference.

**Why it matters**: Deployment is production behavior, not just authoring metadata.

**ZenML gap**: No native Azure managed endpoint primitive.

**Recommended path**:
- Keep deployment Azure-native, or
- Call Azure SDK APIs explicitly from a ZenML step

### AzureML Registry, Responsible AI dashboard, and Designer

**What AzureML does**:
- AzureML Registry supports cross-workspace asset sharing
- Responsible AI dashboard provides Azure-specific evaluation/reporting workflows
- Designer provides visual pipeline authoring

**Why it matters**: These are platform capabilities, not just syntax choices.

**ZenML gap**: No direct equivalent for any of them.

**Recommended path**:
- Keep them Azure-native where required
- Do not claim automatic migration for these features

### Unverified control-flow helpers

Treat these as **unsafe for automatic translation** in this skill:

- `if_else`
- `do_while`
- `parallel_for`
- `set_pipeline_controller_configurations`

**Why it matters**: This skill does not treat them as stable, verified 1:1 migration surfaces.

**Recommended path**:
- Flag them as manual-review-only
- Redesign deliberately using ZenML dynamic pipelines only if the user explicitly accepts a redesign

### Strongly heterogeneous per-node compute

**What AzureML does**: The pipeline DSL can express node-level compute choices cleanly.

**Why it matters**: A literal rewrite can hide the fact that the original pipeline depended on meaningful compute heterogeneity.

**ZenML gap**: ZenML can preserve a lot of Azure compute via the AzureML orchestrator, but per-node heterogeneity is not a simple like-for-like DSL property.

**Recommended path**:
- Consider separate pipelines, step operators, or explicit runtime config
- Flag when compute specialization is core to correctness or cost behavior

---

## Behavioral Differences

### Execution model

| Aspect | AzureML SDK v2 | ZenML |
|---|---|---|
| **Authoring model** | Asset-centric (components, environments, compute, assets) | Python-workflow-centric (`@step`, `@pipeline`) |
| **Main reusable unit** | Registered components and environments | Python code, Docker config, stack config |
| **Execution target** | AzureML jobs | Orchestrator-managed steps, optionally still on AzureML |
| **Run inspection** | AzureML Studio | ZenML dashboard plus backend-specific UI |

### Environment management

| Aspect | AzureML | ZenML |
|---|---|---|
| **Primary primitive** | `Environment(...)` asset | `DockerSettings(...)` |
| **Registry semantics** | Yes | No first-class equivalent |
| **Curated images** | Azure curated environments | Prebuilt parent images |
| **Build context** | Azure-aware environment build | Docker build config in code |

### Compute management

| Aspect | AzureML | ZenML |
|---|---|---|
| **Pipeline-level compute** | `default_compute` | Orchestrator settings |
| **Node-level compute** | `node.compute = ...` | Runtime config / step operator / redesign |
| **Serverless/instance/cluster** | Native Azure modes | Preserved via AzureML orchestrator |
| **Schedule lifecycle** | Azure-managed | ZenML can create schedules, but lifecycle management is limited afterward |

### Data handling

| Aspect | AzureML | ZenML |
|---|---|---|
| **Type surface** | Asset descriptors (`uri_file`, `uri_folder`, `mltable`, model assets) | Python parameters, artifacts, materializers |
| **Primitive outputs** | Limited / asset-oriented | Normal and ergonomic |
| **Versioning story** | Azure asset registry/versioning | Artifact lineage/versioning |
| **Governance boundary** | Asset identity often explicit | Identity often moves into code/config unless preserved deliberately |

### Control flow

| Aspect | AzureML | ZenML |
|---|---|---|
| **Standard DAG edges** | Native | Native |
| **Conditional helpers** | Azure helper APIs exist | Dynamic pipelines / redesign |
| **Loop/fan-out helpers** | Azure helper APIs / parallel abstractions | Dynamic pipelines / `.map()` / redesign |
| **Auto-translation safety** | Mixed | Unsafe for the helper APIs listed above |

### Model management and deployment

| Aspect | AzureML | ZenML |
|---|---|---|
| **Model registration** | Azure model assets | ZenML `Model` plus optional Azure registration |
| **Managed endpoints** | Native Azure deployment primitives | No native equivalent |
| **MLflow** | First-class integration available | Works well inside ZenML steps / experiment tracker |
| **Cross-workspace sharing** | Azure Registry | No direct equivalent |

---

## Migration Decision Tree

Text-based decision procedure for translating AzureML SDK v2 workflows to ZenML.

```text
INPUT: azure_object_type, compute_requirements, environment_usage,
       data_asset_requirements, deployment_requirements, control_flow_helpers

WORKFLOW SHAPE:
  IF object is an @pipeline with standard component edges:
    EMIT @pipeline
    MAP pipeline parameters -> typed Python parameters
  ELSE:
    FLAG MANUAL_ANALYSIS_REQUIRED

COMPONENTS:
  IF component is @command_component:
    EMIT @step
  ELSE IF component is YAML + load_component():
    REWRITE as Python @step
    FLAG warn YAML_COMPONENT_REWRITTEN
  ELSE IF component is registered Azure component:
    FLAG blocker REGISTRY_SEMANTICS_PRESENT
    DECIDE: keep Azure component external OR repackage as Python code

ENVIRONMENTS:
  IF environment is image + conda only:
    MAP -> DockerSettings(parent_image=..., requirements=...)
  ELSE IF environment depends on curated/registered environment semantics:
    MAP runtime intent -> DockerSettings
    FLAG warn ENVIRONMENT_ASSET_SEMANTICS_LOST

COMPUTE:
  IF workflow should keep Azure execution:
    MAP serverless / compute-instance / compute-cluster
      -> AzureMLOrchestratorSettings(mode=...)
  IF node-level heterogeneous compute is important:
    FLAG warn PER_NODE_COMPUTE_REDESIGN
    DECIDE: separate pipelines, step operator, or explicit runtime config

DATA:
  IF inputs/outputs are primitive parameters:
    MAP directly to typed Python params
  IF assets are uri_file / uri_folder / mltable / model assets:
    IF asset identity matters for governance or cross-team reuse:
      KEEP Azure asset reference explicit
      FLAG warn ASSET_IDENTITY_PRESERVED_EXTERNALLY
    ELSE:
      MAP to artifacts or loaded Python objects

ADVANCED JOB TYPES:
  IF workflow uses sweep jobs:
    FLAG blocker SWEEP_REQUIRES_REDESIGN
  IF workflow uses parallel jobs:
    FLAG blocker PARALLEL_JOB_REQUIRES_REDESIGN
  IF workflow uses AutoML:
    FLAG warn AUTOML_EXTERNAL_CALL
    DECIDE: keep Azure AutoML external OR replace with explicit training/HPO

CONTROL FLOW:
  IF helper in {if_else, do_while, parallel_for, set_pipeline_controller_configurations}:
    FLAG blocker CONTROL_FLOW_UNSAFE_FOR_AUTO_TRANSLATION
    REQUIRE manual redesign review

SCHEDULING:
  IF schedule is cron or recurrence AND target is AzureML orchestrator:
    EMIT Schedule(...)
    FLAG warn AZURE_SCHEDULE_LIFECYCLE_MANUAL
  ELSE IF schedule depends on broader Azure schedule management expectations:
    FLAG warn SCHEDULE_GOVERNANCE_REVIEW

DEPLOYMENT:
  IF workflow deploys managed online or batch endpoints:
    FLAG blocker AZURE_DEPLOYMENT_STAYS_NATIVE
    DECIDE: keep Azure-native OR call Azure SDK from a ZenML step

PLATFORM FEATURES:
  IF workflow depends on AzureML Registry, Responsible AI dashboard, or Designer:
    FLAG blocker PLATFORM_FEATURE_NO_ZENML_EQUIVALENT
```

---

## ZenML Features with No AzureML Equivalent

These are capabilities the user gains after migration -- include relevant ones in the "What You Get for Free" section of the migration report.

| ZenML Feature | Description |
|---|---|
| **First-class typed artifacts** | Persist and version rich Python objects, not just Azure asset descriptors. |
| **Step caching** | Skip re-execution when inputs and code have not changed. |
| **Stack abstraction** | Same pipeline code can target local, Kubernetes, Vertex, SageMaker, AzureML, and more by changing the stack. |
| **Model Control Plane** | Manage models, lineage, and promotion workflows beyond raw registry entries. |
| **Service connectors** | Unified cloud authentication patterns with token refresh and secret handling. |
| **Step and pipeline hooks** | Attach cross-cutting logic like notifications without dedicated orchestration nodes. |
| **Project-oriented code reuse** | Reuse Python modules and pipeline code instead of relying on Azure component assets. |
| **Artifact lineage graph** | Track provenance across runs in a way that is more pipeline-native than asset registration alone. |

---

## Anti-Patterns in Migration

| Anti-pattern | Why it's wrong | What to do instead |
|---|---|---|
| Copying AzureML assets into ZenML names 1:1 | Preserves the old mental model but hides architectural changes | Rewrite around `@step`, `@pipeline`, artifacts, and orchestrator settings |
| Treating `Environment(...)` as if it were still a registry-backed object | ZenML does not have that primitive | Translate runtime intent into `DockerSettings` and document the semantic change |
| Converting sweep jobs into Python loops | Changes search execution semantics | Keep Azure sweep native, or redesign around explicit HPO tooling |
| Converting parallel jobs directly into `.map()` | Similar shape, different runtime guarantees | Flag and redesign intentionally |
| Treating MLTable as a native ZenML type | It is not | Keep the Azure boundary explicit or load it deliberately in-step |
| Dropping Azure deployment logic from the migrated workflow | Changes production behavior | Keep endpoints Azure-native or call Azure SDK from steps |
| Pretending `if_else` / `do_while` are safe 1:1 rewrites | This skill does not verify that | Flag them and require manual review |
| Ignoring schedule lifecycle after ZenML creates an Azure schedule | Leads to operational surprises later | Document ownership of updates/deletes in the migration report |
