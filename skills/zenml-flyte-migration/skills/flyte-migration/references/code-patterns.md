# Code Translation Patterns

Side-by-side Flyte → ZenML translations for the patterns most likely to appear in a real migration. Treat these as migration templates, not proof of semantic identity.

ZenML pipelines often materialize important outputs as step artifacts instead of mirroring a Flyte workflow's explicit return value. That is why several ZenML examples below use a `None`-returning pipeline while still preserving the real data flow through step outputs.

## Table of Contents

- [Simple Task/Workflow](#simple-taskworkflow)
- [Dynamic Workflow (`@dynamic`)](#dynamic-workflow-dynamic)
- [Eager Workflow (`@eager`)](#eager-workflow-eager)
- [`map_task()`](#map_task)
- [`conditional()`](#conditional)
- [FlyteFile and FlyteDirectory](#flytefile-and-flytedirectory)
- [StructuredDataset](#structureddataset)
- [ImageSpec and Per-Task Containers](#imagespec-and-per-task-containers)
- [Resources and GPU Requests](#resources-and-gpu-requests)
- [Caching with `cache_version`](#caching-with-cache_version)
- [Retries and Timeouts](#retries-and-timeouts)
- [LaunchPlans and Scheduling](#launchplans-and-scheduling)
- [Raw `ContainerTask`](#raw-containertask)

## Simple Task/Workflow

### Flyte

```python
from flytekit import task, workflow

@task
def extract() -> list[int]:
    return [1, 2, 3]

@task
def total(values: list[int]) -> int:
    return sum(values)

@workflow
def simple_workflow() -> int:
    return total(values=extract())
```

### ZenML

```python
from zenml import pipeline, step

@step
def extract() -> list[int]:
    return [1, 2, 3]

@step
def total(values: list[int]) -> int:
    return sum(values)

@pipeline
def simple_pipeline() -> None:
    total(values=extract())
```

**Key differences:** this is the cleanest case, but Flyte task interfaces and registration semantics are still richer than a bare `@step`.

## Dynamic Workflow (`@dynamic`)

### Flyte

```python
from flytekit import dynamic, task, workflow

@task
def split_items(n: int) -> list[int]:
    return list(range(n))

@task
def process(x: int) -> int:
    return x * 2

@dynamic
def dynamic_body(items: list[int]) -> list[int]:
    return [process(x=i) for i in items]

@workflow
def dynamic_workflow(n: int = 3) -> list[int]:
    return dynamic_body(items=split_items(n=n))
```

### ZenML

```python
from zenml import pipeline, step

@step
def split_items(n: int) -> list[int]:
    return list(range(n))

@step
def process(x: int) -> int:
    return x * 2

@pipeline(dynamic=True)
def dynamic_pipeline(n: int = 3) -> None:
    items = split_items(n=n)
    process.map(x=items)
```

**Migration warning:** do not translate a Flyte dynamic workflow into a static ZenML pipeline with a plain Python loop if the loop count depends on runtime values.

## Eager Workflow (`@eager`)

### Flyte

```python
from flytekit import eager, task

@task
def fetch(x: int) -> int:
    return x + 1

@eager
async def eager_flow(x: int = 1) -> int:
    return await fetch(x=x)
```

### ZenML redesign

```python
from zenml import pipeline, step

@step
def fetch(x: int) -> int:
    return x + 1

@pipeline
def eager_replacement(x: int = 1) -> None:
    fetch(x=x)
```

**Key differences:** `@eager` is not a safe decorator swap. ZenML has no direct equivalent for Flyte's async imperative orchestration style. Treat this as a redesign boundary.

## `map_task()`

### Flyte

```python
from flytekit import map_task, task, workflow

@task
def is_even(x: int) -> bool:
    return x % 2 == 0

@workflow
def parity_workflow(values: list[int]) -> list[bool]:
    return map_task(is_even, concurrency=4)(x=values)
```

### ZenML

```python
from zenml import pipeline, step

@step
def is_even(x: int) -> bool:
    return x % 2 == 0

@pipeline(dynamic=True)
def parity_pipeline(values: list[int]) -> None:
    is_even.map(x=values)
```

**Key differences:** Flyte's `concurrency` and `min_success_ratio` semantics do not carry over automatically. `.map()` is only the closest behavioral neighbour.

## `conditional()`

### Flyte

```python
from flytekit import conditional, task, workflow

@task
def is_positive(x: int) -> bool:
    return x > 0

@task
def pos() -> str:
    return "positive"

@task
def neg() -> str:
    return "non-positive"

@workflow
def conditional_workflow(x: int = 5) -> str:
    flag = is_positive(x=x)
    return (
        conditional("sign")
        .if_(flag.is_true())
        .then(pos())
        .else_()
        .then(neg())
    )
```

### ZenML

```python
from zenml import pipeline, step

@step
def is_positive(x: int) -> bool:
    return x > 0

@step
def pos() -> str:
    return "positive"

@step
def neg() -> str:
    return "non-positive"

@pipeline(dynamic=True)
def conditional_pipeline(x: int = 5) -> None:
    flag = is_positive(x=x)
    if flag.load():
        pos()
    else:
        neg()
```

**Migration warning:** if the original branch relied on Flyte promise semantics deep inside the workflow, verify the dynamic ZenML version carefully.

## FlyteFile and FlyteDirectory

### Flyte

```python
from pathlib import Path
from flytekit import task, workflow
from flytekit.types.directory import FlyteDirectory
from flytekit.types.file import FlyteFile

@task
def write_report() -> FlyteFile:
    path = Path("report.txt")
    path.write_text("hello")
    return FlyteFile(path)

@task
def bundle(report: FlyteFile) -> FlyteDirectory:
    out = Path("bundle")
    out.mkdir(exist_ok=True)
    (out / "report.txt").write_text(Path(report.path).read_text())
    return FlyteDirectory(out)
```

### ZenML

```python
from pathlib import Path
from zenml import pipeline, step

@step
def write_report() -> Path:
    path = Path("report.txt")
    path.write_text("hello")
    return path

@step
def bundle(report: Path) -> Path:
    out = Path("bundle")
    out.mkdir(exist_ok=True)
    (out / "report.txt").write_text(report.read_text())
    return out

@pipeline
def file_pipeline() -> None:
    bundle(report=write_report())
```

**Key differences:** a naive `FlyteFile` → `str` translation is unsafe. If the original workflow depended on remote URI or provenance semantics, use a wrapper type and custom materializer instead of a bare path.

## StructuredDataset

### Flyte

```python
import pandas as pd
from flytekit import task, workflow
from flytekit.types.structured import StructuredDataset

@task
def load_table() -> StructuredDataset:
    df = pd.DataFrame({"x": [1, 2, 3]})
    return StructuredDataset(dataframe=df)

@task
def sum_x(ds: StructuredDataset) -> int:
    df = ds.open(pd.DataFrame).all()
    return int(df["x"].sum())
```

### ZenML

```python
import pandas as pd
from zenml import pipeline, step

@step
def load_table() -> pd.DataFrame:
    return pd.DataFrame({"x": [1, 2, 3]})

@step
def sum_x(df: pd.DataFrame) -> int:
    return int(df["x"].sum())

@pipeline
def dataset_pipeline() -> None:
    sum_x(df=load_table())
```

**Key differences:** Flyte `StructuredDataset` can carry reader / writer / backend metadata. Add validation or metadata logging if that mattered in the source workflow.

## ImageSpec and Per-Task Containers

### Flyte

```python
from flytekit import ImageSpec, task

train_image = ImageSpec(
    name="trainer",
    packages=["scikit-learn==1.5.1", "pandas==2.2.2"],
)

@task(container_image=train_image)
def train() -> str:
    return "trained"
```

### ZenML

```python
from zenml import step
from zenml.config.docker_settings import DockerSettings

train_settings = DockerSettings(
    requirements=["scikit-learn==1.5.1", "pandas==2.2.2"],
)

@step(settings={"docker": train_settings})
def train() -> str:
    return "trained"
```

**Key differences:** the intent maps well, but the image build lifecycle and reproducibility story are not identical. Validate the final image story explicitly.

## Resources and GPU Requests

### Flyte

```python
from flytekit import Resources, task

@task(
    requests=Resources(cpu="2", mem="4Gi", gpu="1"),
    limits=Resources(cpu="4", mem="8Gi", gpu="1"),
)
def train() -> str:
    return "trained"
```

### ZenML

```python
from zenml import step
from zenml.config.resource_settings import ResourceSettings

@step(settings={"resources": ResourceSettings(cpu_count=4, memory="8Gi", gpu_count=1)})
def train() -> str:
    return "trained"
```

**Key differences:** Flyte separates requests and limits more explicitly. ZenML exposes portable hints whose final enforcement depends on the target stack.

## Caching with `cache_version`

### Flyte

```python
from flytekit import task

@task(cache=True, cache_version="v2")
def expensive_square(x: int) -> int:
    return x * x
```

### ZenML

```python
from zenml import step

@step(enable_cache=True)
def expensive_square(x: int, cache_buster: str = "v2") -> int:
    return x * x
```

**Key differences:** the safe migration is an explicit cache-buster parameter or config value. Do not claim a first-class identical `cache_version` field unless you have verified it for the target ZenML version.

## Retries and Timeouts

### Flyte

```python
from datetime import timedelta
from flytekit import task

@task(retries=3, timeout=timedelta(minutes=10))
def flaky_step() -> str:
    return "ok"
```

### ZenML

```python
from zenml import step
from zenml.config.retry_config import StepRetryConfig

@step(retry=StepRetryConfig(max_retries=3, delay=60, backoff=2))
def flaky_step() -> str:
    return "ok"
```

**Key differences:** retries map fairly well. Timeout portability does not. Treat timeout migration as orchestrator-specific unless you have verified the target backend.

## LaunchPlans and Scheduling

### Flyte

```python
from datetime import timedelta
from flytekit import FixedRate, LaunchPlan, task, workflow

@task
def train(learning_rate: float, epochs: int) -> str:
    return f"lr={learning_rate}, epochs={epochs}"

@workflow
def training_workflow(learning_rate: float = 0.01, epochs: int = 5) -> str:
    return train(learning_rate=learning_rate, epochs=epochs)

daily_lp = LaunchPlan.get_or_create(
    name="daily-training",
    workflow=training_workflow,
    default_inputs={"learning_rate": 0.01},
    fixed_inputs={"epochs": 10},
    schedule=FixedRate(duration=timedelta(days=1)),
)
```

### ZenML

```python
from zenml import pipeline, step
from zenml.config.schedule import Schedule

@step
def train(learning_rate: float, epochs: int) -> str:
    return f"lr={learning_rate}, epochs={epochs}"

@pipeline
def training_pipeline(learning_rate: float = 0.01, epochs: int = 5) -> None:
    train(learning_rate=learning_rate, epochs=epochs)

if __name__ == "__main__":
    daily = Schedule(cron_expression="0 0 * * *")
    training_pipeline.with_options(schedule=daily)(learning_rate=0.01, epochs=10)
```

**Key differences:** the schedule translates, but `LaunchPlan` is much more than a schedule. `default_inputs`, `fixed_inputs`, notifications, and execution identity all need explicit redesign.

## Raw `ContainerTask`

### Flyte

```python
from flytekit import workflow
from flytekit.core.container_task import ContainerTask
from flytekit.types.file import FlyteFile
from flytekit import kwtypes

sed_task = ContainerTask(
    name="sed-task",
    image="bash:5.2",
    command=["/bin/sh", "-c", "sed 's/foo/bar/g' {{.inputs.infile}} > {{.outputs.out}}"],
    inputs=kwtypes(infile=FlyteFile),
    outputs=kwtypes(out=FlyteFile),
)

@workflow
def container_workflow(infile: FlyteFile) -> FlyteFile:
    return sed_task(infile=infile)
```

### ZenML redesign

```python
from pathlib import Path
from zenml import pipeline, step

@step
def run_container_job(infile: Path) -> Path:
    # TODO(migration): replace with explicit job submission, IO staging,
    # and output collection. ZenML has no first-class ContainerTask analogue.
    raise NotImplementedError

@pipeline
def container_pipeline(infile: Path) -> None:
    run_container_job(infile=infile)
```

**Migration warning:** Flyte `ContainerTask` is a first-class raw-container primitive with typed file-based IO. ZenML has no safe direct equivalent. Treat this as a redesign boundary, not a code translation exercise.
