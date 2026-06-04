# Dynamic Pipelines Reference

Dynamic pipelines (`@pipeline(dynamic=True)`) build their DAG at runtime, allowing Python control flow (loops, conditionals) driven by step output values.

## When to use

- The number of steps depends on data computed during execution
- You need fan-out/fan-in (map/reduce) patterns
- Conditional execution based on intermediate results
- Agentic workflows where behavior is determined incrementally

For fixed-topology ML DAGs (ingest → preprocess → train → evaluate), use static pipelines.

## Orchestrator support

| Orchestrator | Isolated steps | Handles orchestration env failures |
|---|:---:|:---:|
| LocalOrchestrator | No | No |
| LocalDockerOrchestrator | No | No |
| KubernetesOrchestrator | Yes | Yes |
| VertexOrchestrator | Yes | No |
| SagemakerOrchestrator | Yes | No |
| AzureMLOrchestrator | Yes | No |

## Execution modes and failure behavior

Dynamic pipelines default to `STOP_ON_FAILURE`. They currently support:

- `STOP_ON_FAILURE`: the default and safest choice.
- `FAIL_FAST`: supported, but already-running inline steps may finish before the pipeline stops; isolated steps can be shut down immediately.
- `CONTINUE_ON_FAILURE`: not supported for dynamic pipelines. ZenML warns and uses `STOP_ON_FAILURE` instead. If the workflow really needs to continue after a known failure, catch that exception in pipeline logic and make the fallback explicit.

## Step runtime modes

```python
@step(runtime="isolated")   # Separate container/process (better isolation, parallel-capable)
def heavy_step() -> None: ...

@step(runtime="inline")     # Same process as orchestrator (faster, no container overhead)
def lightweight_step() -> None: ...
```

Use `runtime="isolated"` for parallel steps and resource isolation. Use `runtime="inline"` for fast, sequential steps.

## Artifact name substitutions

Dynamic pipelines support artifact name substitutions in the same way as regular pipelines:

```python
from typing import Annotated
from zenml import ArtifactConfig, pipeline, step

@step(substitutions={"suffix": "validated"})
def produce() -> Annotated[int, ArtifactConfig(name="score_{suffix}")]:
    return 1

@step
def consume(score: int) -> None:
    print(score)

@pipeline(dynamic=True)
def dynamic_pipeline() -> None:
    score = produce()
    consume(score)
```

Caveat: `child_pipeline.embed(...)` does not apply the child pipeline's own configuration. If the child defines substitutions, tags, settings, hooks, or other pipeline-level config, that config is ignored when embedded; the parent run's config governs the inline steps.

## `.load()` vs `.chunk()` — the critical distinction

| Method | Returns | Use for |
|--------|---------|---------|
| `.load()` | Actual Python data | Decisions, control flow, iteration |
| `.chunk(index=i)` | A DAG edge reference | Wiring to downstream steps |

**Mental model**: `.load()` is for decisions (gets values for Python logic). `.chunk()` is for wiring (tells the orchestrator "this step depends on item X from upstream").

```python
@pipeline(dynamic=True)
def filtered_pipeline() -> None:
    items = produce_list()

    for index, value in enumerate(items.load()):   # load: iterate + filter
        if value > 0:
            chunk = items.chunk(index=index)       # chunk: wire DAG edge
            process(chunk)
```

For large artifacts, prefer passing futures/artifacts directly to downstream steps instead of `.load()` when you do not need Python-side control flow.

## Map/reduce with `.map()`

Fan out a step over a collection:

```python
from zenml import pipeline, step

@step
def producer() -> list[int]:
    return [1, 2, 3]

@step
def worker(value: int) -> int:
    return value * 2

@step
def reducer(values: list[int]) -> int:
    return sum(values)

@pipeline(dynamic=True)
def map_reduce():
    values = producer()
    results = worker.map(values)   # Creates 3 worker steps
    reducer(results)               # Receives list of artifacts
```

`.map()` input sources:
- A single list-like output artifact
- A list of output artifacts
- The output of a `.map()` or `.product()` call (if the mapped/product step has a single output)

## Cartesian product with `.product()`

Create a step for every combination of elements across input sequences:

```python
@pipeline(dynamic=True)
def cartesian():
    a = int_values()    # [1, 2]
    b = str_values()    # ["a", "b", "c"]
    do_something.product(a=a, b=b)   # 2 * 3 = 6 steps
```

## Broadcasting with `unmapped()`

Pass a sequence-like artifact as a whole to each mapped invocation (avoid splitting):

```python
from zenml import unmapped

@pipeline(dynamic=True)
def broadcast_example():
    items = producer(length=3)
    config = producer(length=4)
    consumer.map(a=items, b=unmapped(config))   # Each call gets full config list
```

## Unpacking multi-output maps with `.unpack()`

If a mapped step returns multiple outputs, split them into separate lists:

```python
results = compute.map(a=ints)     # compute returns tuple[int, int]
double, triple = results.unpack() # Split by output

doubles = [f.load() for f in double]  # [2, 4]
triples = [f.load() for f in triple]  # [3, 6]
```

`unpack()` works for both `.map()` and `.product()` results.

## Parallel execution with `.submit()`

`.submit()` returns a `StepRunFuture` for non-blocking execution:

```python
@pipeline(dynamic=True)
def parallel_pipeline():
    futures = []
    for i in range(5):
        future = heavy_step.submit(arg=i)   # Non-blocking
        futures.append(future)

    # Wait and get data
    result = futures[0].load()

    # Or pass to downstream step (auto-waits)
    reducer(futures[0])
```

`StepRunFuture` methods:
- `.result()` — Wait and return the artifact response(s)
- `.load()` — Wait and load the actual data
- Pass directly to a step — ZenML auto-waits for it

With `runtime="isolated"`, submitted steps run in separate containers. With `runtime="inline"`, they run in separate threads.

## Child pipelines inside dynamic pipelines

Dynamic pipelines can call other dynamic pipelines from inside the pipeline body:

```python
from zenml import pipeline, step

@step
def produce_number() -> int:
    return 42

@pipeline(dynamic=True)
def child_pipeline():
    return produce_number()

@step
def consume_number(value: int) -> None:
    print(value)

@pipeline(dynamic=True)
def parent_pipeline():
    child_output = child_pipeline()  # synchronous child run
    consume_number(child_output)
```

Key behavior:
- Only dynamic pipelines can be called as child pipelines.
- Child pipelines run on the same stack as the parent.
- Child calls are allowed in pipeline bodies, not inside step functions.
- Child outputs can be `None`, a single artifact, or a tuple of artifacts, and can be passed directly to downstream steps.

### `child_pipeline.submit(...)`

Use `.submit(...)` for a concurrent child run:

```python
@pipeline(dynamic=True)
def parent_pipeline_concurrent():
    future = child_pipeline.submit()
    child_output = future.result()
    consume_number(child_output)
```

### `child_pipeline.embed(...)`

Use `.embed(...)` to reuse the child pipeline body inside the parent run without creating a separate child run:

```python
@pipeline(dynamic=True)
def parent_pipeline_inline():
    child_output = child_pipeline.embed()
    consume_number(child_output)
```

Embedded child pipelines are useful for code reuse, but they are not failure-isolated and do not get their own dashboard run.

`embed(...)` limitations:
- The child pipeline's own config is ignored: settings, retry, cache/log flags, environment, secrets, tags, substitutions, model, hooks, and `depends_on` templates.
- Per-step Docker overrides declared through the child pipeline are ignored; the parent image is used for inline isolated steps.
- Exceptions in the embedded body abort the parent run.

If those boundaries matter, call `child_pipeline(...)` or `child_pipeline.submit(...)` instead so ZenML creates a real child run.

## Build, code, Docker, and token inheritance

Child runs share the parent's orchestration environment, image, and source bundle:

- No new Docker build is triggered for child calls.
- The child snapshot inherits the parent's build, code reference, and code path.
- The child's code and dependencies must already be available in the parent image.
- Pipeline-level Docker settings on the child are overridden by the parent's settings.
- Nested runs share the parent's API-token lineage; automation built around run-scoped tokens should assume access is limited to the root run and descendants, not sibling or unrelated runs.

This applies to `child_pipeline(...)`, `child_pipeline.submit(...)`, and `child_pipeline.embed(...)`.

## Resume/idempotency caveat

Child run identity depends on call order. If the first `my_child()` call becomes `pipeline:my_child` and the second becomes `pipeline:my_child_2`, inserting or reordering calls before them shifts those IDs. On resume, shifted IDs can cause previously completed child runs to execute again. The same caveat applies to step invocation IDs.

## YAML config for dynamic pipelines

Use `depends_on` to make steps configurable via YAML:

```python
@pipeline(dynamic=True, depends_on=[some_step])
def my_pipeline():
    some_step()
```

```yaml
steps:
  some_step:
    parameters:
      arg: 3
```

## Server-triggered runs

```python
from zenml.client import Client

Client().trigger_pipeline(
    snapshot_name_or_id=<ID>,
    run_configuration={"parameters": {"my_param": 3}},
)
```

## Limitations

- `CONTINUE_ON_FAILURE` is not supported for dynamic pipelines; ZenML falls back to `STOP_ON_FAILURE`.
- `FAIL_FAST` does not immediately cancel already-running inline steps; isolated steps can be shut down.
- Logging is not threadsafe for concurrent steps, so logs may interleave.
- `.map()` only works over artifacts from the same pipeline run, not raw data or external artifacts.
- Chunk size for mapped collection loading defaults to 1 and is not yet configurable.
- A failure in one submitted step does not automatically stop all other already-running work unless the execution mode and runtime allow it.

