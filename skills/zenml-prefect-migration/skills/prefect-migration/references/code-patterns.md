# Code Translation Patterns

Side-by-side Prefect → ZenML translations for the migration patterns that come up most often.

## Table of Contents

- [Simple Linear Flow](#simple-linear-flow)
- [Dynamic Branching on Task Results](#dynamic-branching-on-task-results)
- [Runtime Fan-out with `.submit()` / `.map()`](#runtime-fan-out-with-submit--map)
- [Nested Flows](#nested-flows)
- [Retries](#retries)
- [Failure-as-Data Redesign](#failure-as-data-redesign)
- [Blocks and Secrets](#blocks-and-secrets)
- [Scheduling and Deployments](#scheduling-and-deployments)

---

## Simple Linear Flow

### Prefect

```python
from prefect import flow, task

@task
def extract() -> list[int]:
    return [1, 2, 3]

@task
def transform(xs: list[int]) -> list[int]:
    return [x * 10 for x in xs]

@task
def load(xs: list[int]) -> None:
    print(xs)

@flow
def etl() -> None:
    load(transform(extract()))
```

### ZenML

```python
from typing import List
from zenml import pipeline, step

@step
def extract() -> List[int]:
    return [1, 2, 3]

@step
def transform(xs: List[int]) -> List[int]:
    return [x * 10 for x in xs]

@step
def load(xs: List[int]) -> None:
    print(xs)

@pipeline
def etl() -> None:
    load(transform(extract()))
```

**Why this is safe**: the workflow shape is static and Prefect tasks are only being used for dataflow, not runtime orchestration tricks.

---

## Dynamic Branching on Task Results

### Prefect

```python
from prefect import flow, task

@task
def score(text: str) -> int:
    return len(text)

@task
def long_path(text: str) -> None:
    print(text.upper())

@task
def short_path(text: str) -> None:
    print(text.lower())

@flow
def route(text: str) -> None:
    s = score(text)
    if s > 5:
        long_path(text)
    else:
        short_path(text)
```

### ZenML static rewrite — only valid if the branch is really a parameter

```python
from zenml import pipeline, step

@step
def long_path(text: str) -> None:
    print(text.upper())

@step
def short_path(text: str) -> None:
    print(text.lower())

@pipeline
def route(text: str, use_long_path: bool) -> None:
    if use_long_path:
        long_path(text)
    else:
        short_path(text)
```

### ZenML dynamic rewrite — when the branch truly depends on a step output

```python
from zenml import pipeline, step

@step
def score(text: str) -> int:
    return len(text)

@step
def long_path(text: str) -> None:
    print(text.upper())

@step
def short_path(text: str) -> None:
    print(text.lower())

@pipeline(dynamic=True)
def route(text: str) -> None:
    decision = score(text)
    if decision.load() > 5:
        long_path(text)
    else:
        short_path(text)
```

**Migration warning**: if the original Prefect flow relied on runtime branching, do not silently downgrade it to a static parameter branch.

---

## Runtime Fan-out with `.submit()` / `.map()`

### Prefect

```python
from prefect import flow, task
from prefect.task_runners import ThreadPoolTaskRunner

@task
def discover_files() -> list[str]:
    return ["a.csv", "b.csv", "c.csv"]

@task
def process_file(path: str) -> str:
    return path.upper()

@task
def summarize(items: list[str]) -> None:
    print(items)

@flow(task_runner=ThreadPoolTaskRunner(max_workers=4))
def batch_flow() -> None:
    files = discover_files()
    futures = [process_file.submit(path) for path in files]
    summarize([f.result() for f in futures])
```

### ZenML dynamic pipeline

```python
from typing import List
from zenml import pipeline, step

@step
def discover_files() -> List[str]:
    return ["a.csv", "b.csv", "c.csv"]

@step
def process_file(path: str) -> str:
    return path.upper()

@step
def summarize(items: List[str]) -> None:
    print(items)

@pipeline(dynamic=True)
def batch_flow() -> None:
    files = discover_files()
    processed = process_file.map(files)
    summarize(processed)
```

**Migration warning**: this preserves the fan-out shape, not the full Prefect task-runner semantics. If correctness depends on thread/process/Dask/Ray behavior, flag it for review.

---

## Nested Flows

### Prefect

```python
from prefect import flow, task

@task
def prepare(x: int) -> int:
    return x + 1

@flow
def child_flow(x: int) -> int:
    return prepare(x)

@flow
def parent_flow(x: int) -> int:
    return child_flow(x) * 2
```

### ZenML

```python
from zenml import pipeline, step

@step
def prepare(x: int) -> int:
    return x + 1

def child_component(x: int):
    return prepare(x)

@step
def double(x: int) -> int:
    return x * 2

@pipeline
def parent_pipeline(x: int) -> None:
    child_result = child_component(x)
    double(child_result)
```

**Migration warning**: this preserves composition, not Prefect's nested flow-run tracking or child-flow state semantics.

---

## Retries

### Prefect

```python
from prefect import flow, task

@task(retries=3, retry_delay_seconds=10)
def flaky() -> str:
    ...

@flow
def train() -> str:
    return flaky()
```

### ZenML

```python
from zenml import pipeline, step
from zenml.config.retry_config import StepRetryConfig

RETRY = StepRetryConfig(max_retries=3, delay=10)

@step(retry=RETRY)
def flaky() -> str:
    ...

@pipeline
def train() -> None:
    flaky()
```

**Migration warning**: Prefect retry conditions, jitter, or state-aware retry logic may need more than a simple `StepRetryConfig`.

---

## Failure-as-Data Redesign

Use this when the Prefect flow relied on `return_state=True` or `allow_failure()`.

### Prefect

```python
from prefect import flow, task
from prefect.utilities.annotations import allow_failure

@task
def might_fail(x: int) -> int:
    if x < 0:
        raise ValueError("negative")
    return x * 2

@task
def inspect(result) -> str:
    return str(result)

@flow
def f(x: int) -> str:
    state = might_fail.submit(x, return_state=True)
    return inspect.submit(allow_failure(state)).result()
```

### ZenML redesign

```python
from typing import Any, Dict
from zenml import pipeline, step

@step
def might_fail(x: int) -> Dict[str, Any]:
    try:
        if x < 0:
            raise ValueError("negative")
        return {"ok": True, "value": x * 2, "error": None}
    except Exception as exc:  # replace with narrower exception if known
        return {"ok": False, "value": None, "error": str(exc)}

@step
def inspect(result: Dict[str, Any]) -> str:
    if result["ok"]:
        return f"value={result['value']}"
    return f"error={result['error']}"

@pipeline
def f(x: int) -> None:
    inspect(might_fail(x))
```

**Migration warning**: this is a redesign, not a direct translation. The point is to make failure handling explicit in the data model.

---

## Blocks and Secrets

### Prefect

```python
from prefect import flow, task
from prefect.blocks.system import Secret

@task
def call_api() -> None:
    token = Secret.load("api-token").get()
    print(token[:3])

@flow
def api_flow() -> None:
    call_api()
```

### ZenML

```python
from zenml import pipeline, step
from zenml.client import Client

@step
def call_api(secret_name: str = "api-token") -> None:
    secret = Client().get_secret(secret_name)
    # Migration note: this example assumes the ZenML secret stores a field
    # named `token`. If the original Prefect block held a different schema,
    # migrate that schema explicitly instead of guessing.
    token = secret.secret_values["token"]
    print(token[:3])

@pipeline
def api_flow() -> None:
    call_api()
```

**Migration warning**: a Prefect Block may also carry schema, methods, or mixed config. Secret-only blocks migrate cleanly; richer blocks should be decomposed.

---

## Scheduling and Deployments

### Prefect

```python
from prefect import flow

@flow
def daily_flow(customer: str = "acme") -> None:
    ...

if __name__ == "__main__":
    daily_flow.deploy(
        name="daily-acme",
        work_pool_name="k8s-pool",
        cron="0 2 * * *",
        parameters={"customer": "acme"},
    )
```

### ZenML

```python
from zenml import pipeline
from zenml.config.schedule import Schedule

@pipeline
def daily_flow(customer: str = "acme") -> None:
    ...

schedule = Schedule(cron_expression="0 2 * * *")
daily_flow = daily_flow.with_options(schedule=schedule)
daily_flow(customer="acme")
```

**Migration warning**:
- Prefect Deployment ≠ ZenML pipeline deployment.
- Work pools do not have a 1:1 ZenML OSS equivalent.
- The actual migration target is usually: schedule + orchestrator + YAML/runtime config.
