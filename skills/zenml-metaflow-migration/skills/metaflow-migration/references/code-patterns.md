# Code Translation Patterns

Side-by-side Metaflow -> ZenML translations for the patterns that show up most often in real migrations. Each section includes the source pattern, a ZenML sketch, the key semantic differences, and a migration warning when a silent approximation would be dangerous.

## Table of Contents

- [Simple Linear Flow](#simple-linear-flow)
- [Data Passing via `self` Attributes](#data-passing-via-self-attributes)
- [Parameter Handling](#parameter-handling)
- [IncludeFile and External Inputs](#includefile-and-external-inputs)
- [Branch and Join](#branch-and-join)
- [Foreach Fan-Out and Reduce](#foreach-fan-out-and-reduce)
- [Retry, Catch, and Timeout](#retry-catch-and-timeout)
- [Resources and Remote Compute](#resources-and-remote-compute)
- [Environment and Dependency Management](#environment-and-dependency-management)
- [Scheduling](#scheduling)
- [Resume and Caching](#resume-and-caching)
- [`current` Runtime Context](#current-runtime-context)
- [Client API and Historical Lookups](#client-api-and-historical-lookups)

---

## Simple Linear Flow

### Metaflow

```python
from metaflow import FlowSpec, step


class LinearFlow(FlowSpec):
    @step
    def start(self):
        self.x = 1
        self.next(self.step_a)

    @step
    def step_a(self):
        self.y = self.x + 1
        self.next(self.end)

    @step
    def end(self):
        print(self.y)


if __name__ == "__main__":
    LinearFlow()
```

### ZenML

```python
from typing import Annotated

from zenml import pipeline, step


@step
def start() -> Annotated[int, "x"]:
    return 1


@step
def step_a(x: int) -> Annotated[int, "y"]:
    return x + 1


@step
def end(y: int) -> None:
    print(y)


@pipeline
def linear_pipeline() -> None:
    x = start()
    y = step_a(x)
    end(y)
```

**Key differences**
- `FlowSpec` becomes a `@pipeline`.
- `self.*` becomes explicit step outputs.
- `start` and `end` are just names in ZenML, not reserved nodes.

**Migration warning**
- Do not translate `self.x` into mutable global state. In Metaflow it is persisted; in ZenML only returned values are persisted.

---

## Data Passing via `self` Attributes

### Metaflow

```python
from metaflow import FlowSpec, step


class FeaturesFlow(FlowSpec):
    @step
    def start(self):
        self.raw = [1, 2, 3]
        self.next(self.transform)

    @step
    def transform(self):
        self.features = [x * 10 for x in self.raw]
        self.count = len(self.features)
        self.next(self.end)

    @step
    def end(self):
        print(self.features, self.count)
```

### ZenML

```python
from zenml import pipeline, step


@step
def load_raw() -> list[int]:
    return [1, 2, 3]


@step
def transform(raw: list[int]) -> tuple[list[int], int]:
    features = [x * 10 for x in raw]
    return features, len(features)


@step
def end(features: list[int], count: int) -> None:
    print(features, count)


@pipeline
def features_pipeline() -> None:
    raw = load_raw()
    features, count = transform(raw)
    end(features, count)
```

**Key differences**
- Metaflow can quietly create many artifacts in one step.
- ZenML needs every persisted output to be returned explicitly.

**Migration warning**
- Audit every downstream `self.<name>` read before you choose the ZenML return signature. Missing one output changes behavior silently.

---

## Parameter Handling

### Metaflow

```python
from metaflow import FlowSpec, Parameter, step


class TrainFlow(FlowSpec):
    alpha = Parameter("alpha", default=0.1)

    @step
    def start(self):
        self.lr = self.alpha
        self.next(self.end)

    @step
    def end(self):
        print(self.lr)
```

### ZenML

```python
from zenml import pipeline, step


@step
def choose_lr(alpha: float) -> float:
    return alpha


@step
def end(lr: float) -> None:
    print(lr)


@pipeline
def train_pipeline(alpha: float = 0.1) -> None:
    lr = choose_lr(alpha=alpha)
    end(lr)
```

**Key differences**
- Metaflow parameters are available across the flow as read-only state.
- ZenML parameters enter through the pipeline signature and must be passed where needed.

**Migration warning**
- If the Metaflow flow reads a parameter in many steps, you need to wire it explicitly or bundle it into a config object.

---

## IncludeFile and External Inputs

### Metaflow

```python
from metaflow import FlowSpec, IncludeFile, step


class ConfigFlow(FlowSpec):
    prompt = IncludeFile("prompt", default="prompt.txt")

    @step
    def start(self):
        self.prompt_text = self.prompt
        self.next(self.end)

    @step
    def end(self):
        print(self.prompt_text[:20])
```

### ZenML

```python
from pathlib import Path

from zenml import pipeline, step


@step
def ingest_prompt(path: str) -> str:
    return Path(path).read_text()


@step
def end(prompt_text: str) -> None:
    print(prompt_text[:20])


@pipeline
def config_pipeline(prompt_path: str = "prompt.txt") -> None:
    prompt_text = ingest_prompt(prompt_path)
    end(prompt_text)
```

**Key differences**
- Metaflow `IncludeFile` makes file contents a built-in flow input.
- ZenML expects explicit ingestion or external artifact registration.

**Migration warning**
- Be explicit about whether the file contents, the file path, or a registered external artifact is the true contract.

---

## Branch and Join

### Metaflow

```python
from metaflow import FlowSpec, step


class BranchFlow(FlowSpec):
    @step
    def start(self):
        self.value = 5
        self.next(self.left, self.right)

    @step
    def left(self):
        self.score = self.value + 1
        self.next(self.join)

    @step
    def right(self):
        self.score = self.value + 10
        self.next(self.join)

    @step
    def join(self, inputs):
        self.best = max(inp.score for inp in inputs)
        self.next(self.end)

    @step
    def end(self):
        print(self.best)
```

### ZenML

```python
from zenml import pipeline, step


@step
def start() -> int:
    return 5


@step
def left(value: int) -> int:
    return value + 1


@step
def right(value: int) -> int:
    return value + 10


@step
def join(left_score: int, right_score: int) -> int:
    return max(left_score, right_score)


@step
def end(best: int) -> None:
    print(best)


@pipeline
def branch_pipeline() -> None:
    value = start()
    left_score = left(value)
    right_score = right(value)
    best = join(left_score=left_score, right_score=right_score)
    end(best)
```

**Key differences**
- Metaflow join steps receive a special `inputs` object.
- ZenML join steps must name every incoming branch artifact explicitly.

**Migration warning**
- If the original join relied on implicit propagation or `merge_artifacts(inputs)`, you must redesign the join contract instead of guessing.

---

## Foreach Fan-Out and Reduce

### Metaflow

```python
from metaflow import FlowSpec, step


class BatchFlow(FlowSpec):
    @step
    def start(self):
        self.items = ["a", "b", "c"]
        self.next(self.process, foreach="items")

    @step
    def process(self):
        self.result = self.input.upper()
        self.next(self.join)

    @step
    def join(self, inputs):
        self.results = [inp.result for inp in inputs]
        self.next(self.end)

    @step
    def end(self):
        print(self.results)
```

### ZenML

```python
from zenml import pipeline, step


@step
def list_items() -> list[str]:
    return ["a", "b", "c"]


@step
def process_item(item: str) -> str:
    return item.upper()


@step
def collect(results: list[str]) -> None:
    print(results)


@pipeline(dynamic=True)
def batch_pipeline() -> None:
    items = list_items()
    processed = process_item.map(items)
    collect(processed)
```

**Key differences**
- Both express fan-out and reduction, but ZenML does it through dynamic pipelines.
- The current docs list dynamic-pipeline support for `local`, `local_docker`, `kubernetes`, `sagemaker`, `vertex`, and `azureml`.
- When you manually loop in a dynamic pipeline, `.load()` is for making Python decisions and `.chunk(idx)` is for creating DAG edges to individual elements.

**Migration warning**
- Do not rewrite `foreach` as a plain local loop without telling the user. That changes observability, concurrency, and orchestration semantics.

---

## Retry, Catch, and Timeout

### Metaflow

```python
from metaflow import FlowSpec, catch, retry, step, timeout


class RobustFlow(FlowSpec):
    @retry(times=3, minutes_between_retries=1)
    @timeout(seconds=300)
    @catch(var="error")
    @step
    def start(self):
        risky_call()
        self.next(self.end)

    @step
    def end(self):
        print(getattr(self, "error", None))
```

### ZenML

```python
from dataclasses import dataclass

from zenml import pipeline, step
from zenml.config.retry_config import StepRetryConfig


@dataclass
class StepResult:
    ok: bool
    error: str | None


@step(retry=StepRetryConfig(max_retries=3, delay=60, backoff=2))
def guarded_start() -> StepResult:
    try:
        risky_call()
        return StepResult(ok=True, error=None)
    except Exception as exc:  # noqa: BLE001
        return StepResult(ok=False, error=str(exc))


@step
def end(result: StepResult) -> None:
    print(result.error)


@pipeline
def robust_pipeline() -> None:
    result = guarded_start()
    end(result)
```

**Key differences**
- Retry maps well.
- `@catch` does **not**. You must model error-as-data explicitly if you want downstream continuation.
- Timeout behavior is backend-specific in ZenML.

**Migration warning**
- Never claim `@catch` was preserved just because you added `try/except`. The failure semantics changed and must be documented.

---

## Resources and Remote Compute

### Metaflow

```python
from metaflow import FlowSpec, kubernetes, resources, step


class GpuFlow(FlowSpec):
    @kubernetes(cpu=4, memory=16000)
    @resources(gpu=1)
    @step
    def train(self):
        train_model()
        self.next(self.end)
```

### ZenML

```python
from zenml import pipeline, step
from zenml.config.resource_settings import ResourceSettings


@step(
    settings={
        "resources": ResourceSettings(cpu_count=4, memory="16Gi", gpu_count=1),
    }
)
def train() -> None:
    train_model()


@pipeline
def gpu_pipeline() -> None:
    train()
```

**Key differences**
- Resource requests map by intent.
- The actual execution backend is chosen through the stack and orchestrator/step-operator configuration.

**Migration warning**
- Treat `@batch` and backend-specific compute decorators as migration-design questions, not decorator swaps.

---

## Environment and Dependency Management

### Metaflow

```python
from metaflow import FlowSpec, conda, environment, pypi, step


class EnvFlow(FlowSpec):
    @conda(libraries={"pandas": "2.2.0"})
    @pypi(packages={"requests": "2.32.0"})
    @environment(vars={"MODE": "prod"})
    @step
    def start(self):
        ...
```

### ZenML

```python
from zenml import pipeline, step
from zenml.config.docker_settings import DockerSettings


docker_settings = DockerSettings(
    requirements=["pandas==2.2.0", "requests==2.32.0"],
    environment={"MODE": "prod"},
)


@step(settings={"docker": docker_settings})
def start() -> None:
    ...


@pipeline(settings={"docker": docker_settings})
def env_pipeline() -> None:
    start()
```

**Key differences**
- Metaflow often resolves environments per step.
- ZenML models the runtime as a container/image design problem.

**Migration warning**
- Outerbounds Fast Bakery and Metaflow dependency decorators are often best translated at the image level, not step by step.

---

## Scheduling

### Metaflow

```python
from metaflow import FlowSpec, schedule, step


@schedule(cron="0 2 * * *")
class DailyFlow(FlowSpec):
    @step
    def start(self):
        ...
```

### ZenML

```python
from zenml import pipeline, step
from zenml.config.schedule import Schedule


@step
def start() -> None:
    ...


@pipeline
def daily_pipeline() -> None:
    start()


schedule = Schedule(cron_expression="0 2 * * *")
daily_pipeline.with_options(schedule=schedule)()
```

**Key differences**
- Both support scheduling.
- In ZenML, scheduler support depends on the orchestrator.

**Migration warning**
- If the source deployment assumed scheduling was always available, call out the target orchestrator requirement explicitly.

---

## Resume and Caching

### Metaflow

```bash
python my_flow.py resume
```

### ZenML

```python
from zenml import pipeline


@pipeline(enable_cache=True)
def cached_pipeline() -> None:
    ...
```

**Key differences**
- Metaflow `resume` reuses completed work from an interrupted or prior run by flow/task identity.
- ZenML caching reuses step outputs based on code, inputs, settings, and lineage.

**Migration warning**
- Caching is only an approximation of resume. If the source flow depends on exact resume or checkpoint semantics, flag it for review.

---

## `current` Runtime Context

### Metaflow

```python
from metaflow import FlowSpec, current, step


class ContextFlow(FlowSpec):
    @step
    def start(self):
        print(current.run_id, current.step_name)
        self.next(self.end)
```

### ZenML

```python
from zenml import step
from zenml.steps import get_step_context


@step
def start() -> None:
    context = get_step_context()
    print(context.pipeline_run.name, context.step_run.name)
```

**Key differences**
- ZenML exposes runtime metadata, but the available fields are not a full replacement for `current.*`.

**Migration warning**
- If the flow logic depends on Metaflow-specific runtime metadata, make that dependency explicit in the migration report.

---

## Client API and Historical Lookups

### Metaflow

```python
from metaflow import Flow, Run


latest_run = Flow("TrainingFlow").latest_run
artifact = latest_run["score"]["accuracy"].data
```

### ZenML

```python
from zenml.client import Client


client = Client()
pipeline_runs = client.list_pipeline_runs(size=1, sort_by="desc:created")
latest_run = pipeline_runs.items[0]
step_run = next(step for step in latest_run.step_runs.values() if step.name == "score")
```

**Key differences**
- Both expose historical lineage.
- The object graph and traversal APIs differ significantly.

**Migration warning**
- If the flow makes heavy runtime decisions based on historical runs, treat it as a design concern, not just an API translation exercise.
