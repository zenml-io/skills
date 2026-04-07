# Gaps, Flags, and Behavioral Differences

This reference covers the patterns that are most dangerous to silently approximate during migration -- the places where Argo Workflows and ZenML behave fundamentally differently, and where a naive translation changes the pipeline's actual behavior.

## Table of Contents

- [Must-Flag Patterns](#must-flag-patterns)
- [Argo Template Classification Guide](#argo-template-classification-guide)
- [Behavioral Differences](#behavioral-differences)
- [Migration Decision Tree](#migration-decision-tree)
- [ZenML Features with No Argo Equivalent](#zenml-features-with-no-argo-equivalent)

---

## Must-Flag Patterns

These patterns must **never** be silently approximated. Flag them in the migration report and require human review.

### Shared volumes between steps (PVC / `emptyDir`)

**What Argo does**: multiple steps mount the same volume and exchange files directly.

**Why it matters**: the filesystem becomes part of the workflow API. Downstream logic may depend on file layout, mutation order, or large shared intermediates.

**ZenML gap**: ZenML steps are isolated by default. Cross-step local filesystem sharing is not a portable contract.

**Redesign approaches**:
- return typed artifacts or explicit `Path` artifacts
- move state to object storage / database / artifact store
- collapse tightly coupled logic into one step when the shared filesystem is essential

### `containerSet` / multi-container pod

**What Argo does**: multiple containers run in one pod with shared host, network, and filesystem semantics.

**Why it matters**: low-latency sharing and same-pod lifecycle behavior are often part of correctness, not just convenience.

**ZenML gap**: no first-class multi-container same-pod primitive.

**Redesign approaches**:
- collapse the behavior into one step
- manage the multi-container pod from a Kubernetes-coupled step
- keep that subsystem outside ZenML if same-pod semantics are core

### Status-based branching (`depends: "A.Failed"`, `.Errored`, `.Skipped`, ...)

**What Argo does**: controller-level control flow branches on task result states.

**Why it matters**: failure handling, cleanup, and alerting semantics are explicit and often business-critical.

**ZenML gap**: ZenML does not have a portable "run this step iff an upstream step failed" primitive.

**Redesign approaches**:
- model error-as-data explicitly (for example `{ok: bool, result: ...}`)
- branch in dynamic pipelines on explicit status values
- move notifications into hooks
- split recovery / monitoring / cleanup into separate flows

### `onExit`

**What Argo does**: guaranteed finalizer template that runs when the workflow ends, regardless of success or failure.

**Why it matters**: cleanup, notification, and chaining logic often depends on that always-run guarantee.

**ZenML gap**: hooks and execution modes are not equivalent to a workflow-level finalizer.

**Redesign approaches**:
- use idempotent cleanup steps that do not depend on failed outputs
- use `try/finally` inside the owning step where appropriate
- combine hooks, external run monitoring, and careful execution mode choices

### Arbitrary non-Python container steps

**What Argo does**: any container image and command can be the execution primitive.

**Why it matters**: the image itself may carry the entire toolchain and reproducibility story.

**ZenML gap**: ZenML is Python-first; remote execution still expects a Python-capable runtime.

**Redesign approaches**:
- rebuild the tool as Python-callable logic
- create a custom Python-capable base image for ZenML
- keep those nodes outside ZenML if the workflow is fundamentally non-Python

### File-path parameter passing (`valueFrom.path`, `/tmp` contracts)

**What Argo does**: output parameters are often derived from file content; local paths are part of the data contract.

**Why it matters**: hidden path-based contracts break easily when steps become isolated.

**ZenML gap**: ZenML prefers typed returns; filesystem paths are not stable step interfaces.

**Redesign approaches**:
- replace with explicit return values
- return a `Path` artifact when a file contract is genuinely required
- use custom materializers where the contract is file- or archive-shaped

### Synchronization (mutexes / semaphores)

**What Argo does**: controller-managed locks and concurrency controls protect scarce external resources.

**Why it matters**: removing them can create race conditions or overload shared systems.

**ZenML gap**: no general built-in lock primitive.

**Redesign approaches**:
- external lock service (Redis / DB / object-store lock)
- scheduler-level concurrency controls where available
- explicit queueing or batching outside the pipeline

### Sidecars and init containers

**What Argo does**: helper containers run alongside the main workload or prepare the pod before the workload starts.

**Why it matters**: some workflows depend on proxies, local services, or bootstrapping side effects.

**ZenML gap**: not portable; Kubernetes-specific pod customisation may help, but it is not a first-class portable workflow concept.

**Redesign approaches**:
- replace the sidecar dependency with a managed external service
- use Kubernetes-specific orchestrator settings when Kubernetes coupling is acceptable
- bake init-container setup into the image or step preamble when possible

### Dynamic fan-out (`withItems`, `withParam`)

**What Argo does**: controller expands tasks dynamically from literals or JSON strings.

**Why it matters**: cardinality, parallelism, aggregation, and failure behavior are part of the operational contract.

**ZenML gap**: ZenML dynamic fanout uses a different runtime model.

**Redesign approaches**:
- use dynamic pipelines with real list artifacts and `.map()`
- validate orchestrator support and failure semantics
- redesign very large or event-shaped fanout into separate runs if needed

### Argo Events graph

**What Argo does**: EventSource + Sensor + Trigger + dependency logic form a native eventing graph.

**Why it matters**: the event system is part of the platform architecture, not just an optional trigger.

**ZenML gap**: current ZenML docs do not describe an Argo-Events-style native graph. ZenML 0.94.0 introduced a Pro `Trigger` concept, but the first documented trigger type is schedules, and the same 0.94.0 changelog says legacy triggers / actions / event-source APIs were removed.

**Redesign approaches**:
- external event system invokes a ZenML deployment / pipeline run / snapshot trigger
- keep event routing and dependency logic explicit in the surrounding platform
- treat any ZenML trigger usage as a narrow automation target, not as a drop-in EventSource/Sensor replacement

---

## Argo Template Classification Guide

Classify each template before migrating it. This is the Argo analogue of the Databricks notebook guide: it tells you how safe the translation really is.

### Detection checklist

| Pattern | Detection | Risk Level |
|---|---|---|
| Simple `script` template with inline Python | Python logic, clear inputs, clear outputs | LOW -- usually maps to a `@step` |
| Simple `container` template with Python-friendly runtime | CLI or Python code, minimal pod coupling | LOW / MEDIUM -- may map to a `@step` with `DockerSettings` |
| `dag` template using plain success dependencies | `dependencies` only, no status expressions | LOW -- orchestration shape usually maps well |
| `steps` template with simple sequential/parallel groups | Outer sequential arrays, inner parallel arrays | MEDIUM -- structure maps, parallel semantics vary |
| Output parameters from files or `outputs.result` | `valueFrom.path`, stdout/body capture | MEDIUM -- convert to typed returns carefully |
| Artifact passing via file paths | input/output artifacts with explicit paths | MEDIUM -- often becomes `Path` or typed artifacts |
| `resource` or `http` template | API / infrastructure operation wrapped in a template | MEDIUM -- move explicit SDK/client logic into a step |
| `withItems` / `withParam` fanout | loop expansion at runtime | MEDIUM / HIGH -- validate dynamic pipeline semantics |
| `CronWorkflow` | workflow schedule bundled with workflow spec | MEDIUM -- scheduling support is orchestrator-dependent |
| `suspend` | approval or pause node | HIGH -- control-boundary redesign required; only consider `zenml.wait(...)` on recent 0.94.x releases, preferably `>=0.94.1` |
| `containerSet` | multi-container pod | HIGH -- same-pod semantics are core |
| Shared volumes / PVCs / `emptyDir` | cross-step filesystem sharing | HIGH -- redesign hotspot |
| Enhanced `depends` logic | task-status branching | HIGH -- no direct equivalent |
| Sidecars / init containers / daemon containers | extra pod topology | HIGH -- portable translation is not available |
| EventSource / Sensor / Argo Events | event graph outside core workflow | HIGH -- integration architecture decision required |

### Classification outcomes

**Auto-refactorable** (low risk):
- simple `script` template with Python logic
- simple `container` template where a Python-capable ZenML image is reasonable
- plain `dag` or `steps` orchestration using ordinary success-path dependencies
- parameter-only data flow and clear typed return values

**Semi-automatic** (medium risk):
- artifact passing where a file or `Path` contract must be kept explicit
- `resource` / `http` templates
- `withItems`, `withParam`, `withSequence`
- `CronWorkflow`
- pod resource requests and placement settings that can move into orchestrator config

**Manual redesign required** (high risk):
- enhanced `depends` status logic
- `containerSet`
- sidecars / same-pod helper services
- shared PVC / mutable filesystem contracts between steps
- `onExit` when correctness depends on always-run behavior
- synchronization locks
- Argo Events graphs

---

## Behavioral Differences

### Declarative YAML vs imperative Python

| Aspect | Argo Workflows | ZenML Pipelines |
|---|---|---|
| Authoring format | YAML CRDs | Python functions |
| Composition mechanism | Template refs + DAG/steps spec + variable substitution | Function composition and artifact wiring |
| Migration effect | You are porting manifests | You are rewriting the execution program into code |

### File-based artifacts vs typed artifacts

| Aspect | Argo | ZenML |
|---|---|---|
| Data contract | Files, paths, artifact repos | Typed artifacts in an artifact store |
| Output parameters | Often extracted from files / stdout | Explicit return values |
| Migration effect | Path contracts are normal | Typed values are normal; path contracts must be explicit |

### Kubernetes-native vs stack-abstracted

| Aspect | Argo | ZenML |
|---|---|---|
| Execution assumption | Kubernetes is the substrate | Orchestrator is a pluggable stack component |
| Pod-level controls | First-class workflow knobs | Mostly orchestrator-specific settings |
| Migration effect | Kubernetes details sit inside workflow authoring | Separate business logic from infrastructure logic |

### Volume sharing vs container isolation

| Aspect | Argo | ZenML |
|---|---|---|
| Shared filesystems | Common pattern | Not a portable step boundary |
| Same-pod helpers | Sidecars / containerSet / init containers | No portable same-pod equivalent |
| Migration effect | File mutation and pod topology can be part of correctness | Prefer artifacts, explicit stores, or one-step redesign |

### Parameter substitution vs function arguments

| Aspect | Argo | ZenML |
|---|---|---|
| Passing values | `{{...}}` substitution and workflow variables | Typed function args and artifact inputs |
| Produced upstream values | Often still called "parameters" | They become artifacts |
| Migration effect | String-substituted wiring | Explicit type-aware data flow |

### Exit handlers vs hooks and execution modes

| Aspect | Argo | ZenML |
|---|---|---|
| Always-run finalization | `onExit` | No direct run-level `finally` primitive |
| Notifications | Finalizer templates and hooks | Step hooks and external monitoring |
| Migration effect | Cleanup guarantees are stronger in Argo | Finalization must be redesigned explicitly |

---

## Migration Decision Tree

Use this procedure for every Argo component. The goal is to force an explicit decision instead of a hopeful translation.

```text
1. Identify the object kind first.
   - Workflow / WorkflowTemplate / ClusterWorkflowTemplate -> execution logic track
   - CronWorkflow -> split scheduling from execution immediately
   - EventSource / Sensor -> event-integration track, not ordinary workflow translation

2. Identify the template type.
   - container / script / data -> step candidate
   - dag / steps -> orchestration structure to rebuild in Python
   - resource / http / plugin -> explicit SDK/API step
   - suspend / containerSet / sidecars / status-driven control flow -> redesign-first

3. Separate direct inputs from produced data.
   - workflow parameters and literal arguments -> pipeline / step parameters
   - outputs.parameters / outputs.result / outputs.artifacts -> artifacts, not params

4. Ask whether the contract is value-based or path-based.
   - if downstream needs only the value -> return a normal Python type
   - if downstream needs a specific file / archive / URI -> model it explicitly

5. Inspect control flow for anything beyond success-path dependencies.
   - plain dependencies -> usually safe
   - status-based depends / continueOn / onExit / lifecycle hooks -> classify before coding

6. Inspect loop shape.
   - fixed numeric sequence -> Python range(...)
   - literal list or upstream-produced list -> real list artifact + dynamic fanout
   - controller-side status-dependent expansion -> approximate or absent, document the shift

7. Inspect runtime coupling.
   - shared volumes / same-pod services / sidecars / init containers / locks -> redesign territory

8. Move Kubernetes-specific configuration out of business logic.
   - ResourceSettings often survive
   - pod placement, service accounts, tolerations, and init containers belong in orchestrator config

9. Classify the mapping explicitly.
   - Direct = dependency and data semantics stay intact
   - Approximate = intent survives but semantics change
   - Absent = feature depends on controller / pod / event-graph behavior ZenML does not model

10. Only then write the ZenML code.
    - test value correctness
    - test dependency correctness
    - test operational correctness
```

Compact rule of thumb:

```text
If the Argo feature is about values and success-path dependencies, it probably maps.
If it is about filesystems, pod topology, controller policy, or event graphs, it probably needs redesign.
```

---

## ZenML Features with No Argo Equivalent

These are capabilities the user gains after migration. Mention the relevant ones in the "What You Get for Free" section of the migration report.

| ZenML Feature | Description |
|---|---|
| **Typed artifacts with materializers** | Data contracts become explicit Python types instead of file-path conventions. |
| **Artifact versioning and lineage** | Artifacts are versioned and traceable across runs. |
| **Step caching** | Skip re-execution when code and inputs have not changed. |
| **Stack abstraction** | The same pipeline code can move across local, Kubernetes, Kubeflow, Vertex, SageMaker, AzureML, and more. |
| **ExternalArtifact** | First-class injection of pre-existing external data into a typed pipeline graph. |
| **Model Control Plane** | Track model versions, stages, and lineage after ML pipeline migration. |
| **Service connectors** | Unified cloud auth and reusable infrastructure access patterns. |
| **ZenML secrets store** | Workflow-facing secret management that is more centralized at the pipeline platform layer. |
| **YAML run configuration templates** | Multi-environment config layered over one Python codebase. |
| **Snapshots / deployments distinction** | Cleaner separation between batch reruns and long-running serving/inference systems. |
