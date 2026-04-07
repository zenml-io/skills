# Argo Workflows -> ZenML Code Translation Patterns

This reference shows side-by-side translation patterns for the most common Argo migration cases. Use these examples as **shape guides**, not as a promise of perfect 1:1 semantics.

## Table of Contents

- [Simple DAG template with dependencies](#simple-dag-template-with-dependencies)
- [Steps template with sequential and parallel groups](#steps-template-with-sequential-and-parallel-groups)
- [Container template for arbitrary image execution](#container-template-for-arbitrary-image-execution)
- [Script template for inline Python or shell](#script-template-for-inline-python-or-shell)
- [Parameter passing between templates](#parameter-passing-between-templates)
- [Artifact passing between templates](#artifact-passing-between-templates)
- [Conditional execution with `when`](#conditional-execution-with-when)
- [Loops: `withSequence`, `withItems`, `withParam`](#loops-withsequence-withitems-withparam)
- [Retry strategy with backoff](#retry-strategy-with-backoff)
- [Exit handler with `onExit`](#exit-handler-with-onexit)
- [CronWorkflow scheduling](#cronworkflow-scheduling)
- [Shared volume / PVC redesign](#shared-volume--pvc-redesign)
- [ContainerSet redesign](#containerset-redesign)
- [Argo Events integration](#argo-events-integration)

## Simple DAG template with dependencies

### Argo YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: etl-
spec:
  entrypoint: etl
  templates:
    - name: etl
      dag:
        tasks:
          - name: extract
            template: extract
          - name: transform
            dependencies: [extract]
            template: transform
          - name: load
            dependencies: [transform]
            template: load

    - name: extract
      script:
        image: python:3.12
        command: [python]
        source: |
          print("extract")

    - name: transform
      script:
        image: python:3.12
        command: [python]
        source: |
          print("transform")

    - name: load
      script:
        image: python:3.12
        command: [python]
        source: |
          print("load")
```

### ZenML Python

```python
from zenml import pipeline, step


@step
def extract() -> str:
    return "extract"


@step
def transform(value: str) -> str:
    return f"{value}:transform"


@step
def load(value: str) -> None:
    print(f"loading {value}")


@pipeline
def etl_pipeline() -> None:
    raw = extract()
    cooked = transform(raw)
    load(cooked)
```

### Key differences

- Argo dependencies are declared in YAML; ZenML dependencies are created by step calls and artifact wiring.
- The ZenML version makes the data flow explicit instead of pointing to templates by name.

### Migration warning

This mapping stays clean only while the Argo DAG uses ordinary success-path dependencies. If it uses enhanced `depends` logic based on task status, treat it as redesign-first.

## Steps template with sequential and parallel groups

### Argo YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: steps-
spec:
  entrypoint: main
  templates:
    - name: main
      steps:
        - - name: prepare
            template: prepare
        - - name: score-a
            template: score-a
          - name: score-b
            template: score-b
        - - name: combine
            template: combine

    - name: prepare
      script:
        image: python:3.12
        command: [python]
        source: "print('prepare')"
    - name: score-a
      script:
        image: python:3.12
        command: [python]
        source: "print('score-a')"
    - name: score-b
      script:
        image: python:3.12
        command: [python]
        source: "print('score-b')"
    - name: combine
      script:
        image: python:3.12
        command: [python]
        source: "print('combine')"
```

### ZenML Python

```python
from zenml import pipeline, step


@step
def prepare() -> list[int]:
    return [1, 2, 3]


@step
def score_a(data: list[int]) -> int:
    return sum(data)


@step
def score_b(data: list[int]) -> int:
    return max(data)


@step
def combine(a: int, b: int) -> dict[str, int]:
    return {"sum": a, "max": b}


@pipeline
def scoring_pipeline() -> None:
    data = prepare()
    a = score_a(data)
    b = score_b(data)
    combine(a, b)
```

### Key differences

- Argo's outer array means sequence; inner array means potential parallelism.
- In ZenML, concurrency is inferred from the dependency graph and depends on the active orchestrator.

### Migration warning

Do not promise true parallel execution on every orchestrator. The DAG shape maps, but actual parallelism is backend-dependent.

## Container template for arbitrary image execution

### Argo YAML

```yaml
- name: convert
  container:
    image: alpine:3.20
    command: ["sh", "-c"]
    args: ["cat /input/data.txt | tr a-z A-Z > /tmp/out.txt"]
```

### ZenML Python

```python
from pathlib import Path
import subprocess

from zenml import step


@step
def convert(input_path: Path) -> Path:
    # Migration note: Argo container templates can use arbitrary images and
    # commands. This ZenML step assumes the required CLI tools exist inside a
    # Python-capable image.
    output_path = Path("/tmp/out.txt")
    cmd = f"cat {input_path} | tr a-z A-Z > {output_path}"
    subprocess.run(["sh", "-c", cmd], check=True)
    return output_path
```

### Migration warning

If the original container image is the real unit of reproducibility and it is not Python-capable, do not pretend this is a clean migration. Either build a Python-capable image, wrap the tool carefully, or keep that node outside ZenML.

## Script template for inline Python or shell

### Argo YAML

```yaml
- name: train
  script:
    image: python:3.12
    command: [python]
    source: |
      import json
      print(json.dumps({"accuracy": 0.91}))
```

### ZenML Python

```python
from zenml import step


@step
def train() -> dict[str, float]:
    return {"accuracy": 0.91}
```

### Migration warning

Inline Python translates well. Inline shell should usually become `subprocess.run(...)` inside a step, with an explicit note about tool availability and container image requirements.

## Parameter passing between templates

### Argo YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: params-
spec:
  entrypoint: main
  arguments:
    parameters:
      - name: dataset
        value: customers
  templates:
    - name: main
      dag:
        tasks:
          - name: extract
            template: extract
            arguments:
              parameters:
                - name: dataset
                  value: "{{workflow.parameters.dataset}}"
    - name: extract
      inputs:
        parameters:
          - name: dataset
      script:
        image: python:3.12
        command: [python]
        source: |
          print("reading {{inputs.parameters.dataset}}")
```

### ZenML Python

```python
from zenml import pipeline, step


@step
def extract(dataset: str) -> str:
    return f"reading {dataset}"


@pipeline
def extraction_pipeline(dataset: str = "customers") -> None:
    extract(dataset=dataset)
```

### Key differences

- Argo substitutes strings into YAML-managed variables.
- ZenML passes typed Python values through function signatures.

## Artifact passing between templates

### Argo YAML

```yaml
- name: produce-report
  script:
    image: python:3.12
    command: [python]
    source: |
      with open("/tmp/report.json", "w") as f:
          f.write("{\"ok\": true}")
  outputs:
    artifacts:
      - name: report
        path: /tmp/report.json

- name: consume-report
  inputs:
    artifacts:
      - name: report
        path: /tmp/input-report.json
```

### ZenML Python

```python
from pathlib import Path

from zenml import pipeline, step


@step
def produce_report() -> Path:
    report = Path("/tmp/report.json")
    report.write_text('{"ok": true}')
    return report


@step
def consume_report(report: Path) -> dict[str, bool]:
    return {"report_exists": report.exists()}


@pipeline
def report_pipeline() -> None:
    report = produce_report()
    consume_report(report)
```

### Key differences

- Argo treats files and artifact paths as first-class workflow wiring.
- ZenML can preserve a file contract, but it is often better to return a typed object instead.

### Migration warning

Do not preserve a path contract by accident. Use `Path` or URI artifacts only when downstream logic truly depends on a file contract.

## Conditional execution with `when`

### Argo YAML

```yaml
- name: maybe-train
  when: "{{workflow.parameters.should_train}} == true"
  template: train
```

### ZenML Python

```python
from zenml import pipeline, step


@step
def train() -> str:
    return "trained"


@pipeline(dynamic=True)
def training_pipeline(should_train: bool) -> None:
    if should_train:
        train()
```

### Migration warning

This stays relatively clean when the condition is based on input values. If the condition depends on task status or on runtime outputs in a way that changes graph shape, classify it as approximate or absent and explain the semantic shift.

## Loops: `withSequence`, `withItems`, `withParam`

### `withSequence`

#### Argo YAML

```yaml
- name: batch
  template: process
  withSequence:
    count: "3"
```

#### ZenML Python

```python
from zenml import pipeline, step


@step
def process(i: int) -> int:
    return i * 2


@pipeline(dynamic=True)
def loop_pipeline() -> None:
    for i in range(3):
        process(i)
```

### `withItems`

#### Argo YAML

```yaml
- name: greet
  template: hello
  withItems:
    - alex
    - noa
```

#### ZenML Python

```python
from zenml import pipeline, step


@step
def produce_names() -> list[str]:
    return ["alex", "noa"]


@step
def hello(name: str) -> str:
    return f"hello {name}"


@pipeline(dynamic=True)
def greeting_pipeline() -> None:
    names = produce_names()
    hello.map(names)
```

### `withParam`

#### Argo YAML

```yaml
- name: fanout
  template: process
  withParam: "{{steps.make-list.outputs.result}}"
```

#### ZenML Python

```python
from zenml import pipeline, step


@step
def make_list() -> list[dict[str, int]]:
    return [{"value": 1}, {"value": 2}, {"value": 3}]


@step
def process(item: dict[str, int]) -> int:
    return item["value"] * 10


@pipeline(dynamic=True)
def fanout_pipeline() -> None:
    items = make_list()
    process.map(items)
```

### Migration warnings

- Do not preserve `withParam` as "JSON in, JSON out." Return real typed lists.
- Dynamic fanout semantics, failure handling, and concurrency depend on the orchestrator and differ from Argo's controller behavior.

## Retry strategy with backoff

### Argo YAML

```yaml
- name: flaky
  retryStrategy:
    limit: "3"
    backoff:
      duration: "10s"
      factor: 2
```

### ZenML Python

```python
from zenml import step
from zenml.config.retry_config import StepRetryConfig


@step(retry=StepRetryConfig(max_retries=3, delay=10, backoff=2))
def flaky() -> None:
    ...
```

### Migration warning

Argo retry policies and retry expressions are richer than ZenML's portable retry surface. If the original workflow uses policy-specific or expression-based retry behavior, flag it.

## Exit handler with `onExit`

### Argo YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: exit-
spec:
  entrypoint: main
  onExit: finalize
  templates:
    - name: main
      dag:
        tasks:
          - name: work
            template: work
    - name: finalize
      script:
        image: python:3.12
        command: [python]
        source: "print('always run cleanup')"
```

### ZenML Python (redesign sketch)

```python
from zenml import step


@step
def work_with_cleanup() -> None:
    resource = acquire_resource()
    try:
        run_main_work(resource)
    finally:
        cleanup_resource(resource)
```

### Migration warning

Do not equate `onExit` with a success hook, and do not quietly turn failures into `{ok: false}` data without calling that semantic change out explicitly. If the original Argo workflow depends on guaranteed workflow-level finalization, redesign that boundary on purpose:

- move resource ownership into a single step with `try/finally` when possible
- use hooks for notifications, not as finalizer substitutes
- use external cleanup / monitoring when the finalization scope is larger than one step

## CronWorkflow scheduling

### Argo YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: CronWorkflow
metadata:
  name: nightly-report
spec:
  schedule: "0 2 * * *"
  timezone: UTC
  concurrencyPolicy: Forbid
  workflowSpec:
    entrypoint: report
```

### ZenML Python

```python
from zenml import pipeline
from zenml.config.schedule import Schedule


@pipeline
def report_pipeline() -> None:
    ...


schedule = Schedule(cron_expression="0 2 * * *")
report_pipeline.with_options(schedule=schedule)()
```

### Migration warning

Do not assume `concurrencyPolicy: Forbid`, timezone behavior, and schedule lifecycle management map fully to `Schedule(...)`. Scheduling support is orchestrator-dependent, and some CronWorkflow semantics may need extra infrastructure decisions.

## Shared volume / PVC redesign

### Argo YAML

```yaml
spec:
  volumes:
    - name: workspace
      persistentVolumeClaim:
        claimName: shared-data
  templates:
    - name: write
      container:
        image: python:3.12
        volumeMounts:
          - name: workspace
            mountPath: /workspace
    - name: read
      container:
        image: python:3.12
        volumeMounts:
          - name: workspace
            mountPath: /workspace
```

### ZenML Python

```python
from pathlib import Path

from zenml import pipeline, step


@step
def write_dataset() -> Path:
    path = Path("/tmp/dataset.parquet")
    # write file here
    return path


@step
def read_dataset(dataset_path: Path) -> None:
    # consume the path or materialized artifact here
    ...


@pipeline
def pvc_redesign_pipeline() -> None:
    dataset = write_dataset()
    read_dataset(dataset)
```

### Migration warning

This is one of the most important redesign hotspots. Replacing a shared PVC with `/tmp` across multiple ZenML steps is simply wrong. Use typed artifacts, explicit `Path` artifacts, external object storage, or collapse tightly-coupled logic into one step.

## ContainerSet redesign

### Argo YAML

```yaml
- name: main
  containerSet:
    containers:
      - name: downloader
        image: alpine
      - name: processor
        image: python:3.12
        dependencies: [downloader]
```

### ZenML Approach

```python
from zenml import step


@step
def download_and_process() -> None:
    # Migration note: Argo containerSet ran multiple containers in one pod
    # with cheap shared filesystem and network semantics. This redesign keeps
    # the tightly coupled work inside one step boundary.
    ...
```

### Migration warning

Do not split a `containerSet` into several ZenML steps unless you also redesign the data and lifecycle contract. Same pod is not the same as several isolated step containers.

## Argo Events integration

### Argo Concepts

- `EventSource` consumes external events
- `Sensor` applies dependency logic
- `Trigger` submits a workflow or another action

### ZenML Mapping Sketch

```python
from fastapi import FastAPI

app = FastAPI()


@app.post("/trigger")
def trigger_pipeline(payload: dict) -> dict[str, str]:
    # Migration note: external eventing is now the control plane.
    # This endpoint would validate the event and invoke a ZenML deployment,
    # pipeline run, or snapshot trigger through the appropriate API.
    return {"status": "accepted"}
```

### Migration warning

Argo Events is a platform-level event graph, not just a "trigger." In current ZenML docs, the safe story is still external event system -> deployment endpoint / pipeline run / snapshot trigger. ZenML does document a Pro `Trigger` concept, but the first supported trigger type is schedules, and legacy event-source APIs were removed in 0.94.0. Keep the event logic explicit instead of claiming full native parity.
