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

**Important**: Only `STOP_ON_FAILURE` execution mode is supported for dynamic pipelines.

## Step runtime modes

```python
@step(runtime="isolated")   # Separate container/process (better isolation, parallel-capable)
def heavy_step() -> None: ...

@step(runtime="inline")     # Same process as orchestrator (faster, no container overhead)
def lightweight_step() -> None: ...
```

Use `runtime="isolated"` for parallel steps and resource isolation. Use `runtime="inline"` for fast, sequential steps.

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
            chunk = items.chunk(index=index)         # chunk: wire DAG edge
            process(chunk)
```

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
- The output of a `.map()` or `.product()` call (if single output)

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
    run_configuration={"parameters": {"my_param": 3}}
)
```

## Limitations

- Only `STOP_ON_FAILURE` execution mode supported
- Logging not threadsafe for concurrent steps (logs may interleave)
- `.map()` only works over artifacts from the same pipeline run (not external artifacts)
- Chunk size for mapped collection loading defaults to 1, not yet configurable
- A failure in one submitted step does not automatically stop others
