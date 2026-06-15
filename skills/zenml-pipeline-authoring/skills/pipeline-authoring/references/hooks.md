# Lifecycle Hooks

Hooks run custom code at lifecycle points of a step or a run, and record each firing as a queryable `HookInvocation`. Common uses: notify on success or failure, log run details, trigger external workflows, and instrument third-party callbacks.

Older ZenML only supported `on_success` and `on_failure`. The `on_start`, `on_end`, `on_pause`, `on_resume` lifecycle hooks, the `HookInvocation` records, and `run_hook(...)` require a recent ZenML version (0.94+).

## The lifecycle hooks

| Hook | Step scope | Run scope (dynamic pipeline) |
|------|-----------|------------------------------|
| `on_start` | Each execution attempt, before the step body | Once before the run starts |
| `on_end` | Each execution attempt, regardless of outcome | Once when the run reaches a terminal state |
| `on_success` | Once when the step completes successfully | Once when the run completes successfully |
| `on_failure` | Once when the step fails terminally | Once when the run fails |
| `on_pause` | Not available | Once when the run pauses |
| `on_resume` | Not available | Once when a paused run resumes |

Step-level hooks fire for both static and dynamic pipelines, uniformly.

## Static vs dynamic: the trap

The same `@pipeline(on_*=...)` kwarg means two different things depending on whether the pipeline is dynamic. This is the easiest thing to get wrong.

| `@pipeline(on_*=X)` | Static pipeline | Dynamic pipeline |
|---------------------|-----------------|------------------|
| `on_start` / `on_success` / `on_failure` / `on_end` | A per-step default that each step inherits unless it sets its own. Never runs at the pipeline level. | Fires once at the run level. |
| `on_pause` / `on_resume` | Ignored | Fires once at the run level. |

So on a **static** pipeline, `@pipeline(on_failure=notify)` does **not** fire once when the run fails. It attaches `notify` as the default `on_failure` of every step that did not set its own. If you want one notification per run on a static pipeline, attach the hook to a single terminal step instead, or move the pipeline to a dynamic one.

On a **dynamic** pipeline, `@pipeline(on_failure=notify)` fires once at the run level and does not propagate to steps.

## Registering hooks

Pass a callable or a source string to the decorator, `.configure(...)`, or `.with_options(...)`.

```python
from zenml import step, pipeline

def notify_start():
    print("starting")

def notify_end(exception=None):
    print("finished")

@step(on_start=notify_start, on_end=notify_end)
def my_step() -> int:
    return 42

@pipeline(on_start=notify_start, on_end=notify_end)
def my_pipeline():
    my_step()

# Override at configuration time
my_step = my_step.with_options(on_failure="my_module.alert_on_failure")
```

## Hook signatures

`on_start`, `on_success`, `on_pause`, `on_resume` take no arguments. `on_failure` and `on_end` optionally take a single `BaseException`.

```python
from typing import Optional

def on_end(): ...
def on_end(exception: Optional[BaseException] = None): ...
```

`exception` is set only when the attempt or run failed. Read details about the current step or run from the context.

```python
from zenml import get_step_context, step

def on_failure(exception: BaseException):
    context = get_step_context()
    print(f"Failed step: {context.step_run.name}")

@step(on_failure=on_failure)
def my_step(some_parameter: int = 1):
    raise ValueError("My exception")
```

Run-scope hooks on a dynamic pipeline fire outside any step. Read the run from the run context instead, via `DynamicPipelineRunContext.get().run`.

Hook functions can be `async def`. ZenML runs the coroutine to completion and blocks until it finishes.

## Sending alerts from hooks

ZenML ships built-in alerter hooks that post to the active stack's alerter.

```python
from zenml.hooks import alerter_failure_hook, alerter_success_hook

@step(on_failure=alerter_failure_hook, on_success=alerter_success_hook)
def my_step():
    ...
```

For custom logic, write a plain function and call the alerter yourself.

```python
from zenml import get_step_context
from zenml.client import Client

def on_failure():
    step_name = get_step_context().step_run.name
    Client().active_stack.alerter.post(f"{step_name} just failed!")
```

## Behavior notes

- **Retries.** A retried step fires one `on_start` / `on_end` pair per attempt. `on_success` and `on_failure` fire exactly once, at the terminal outcome.
- **Cache hits.** A cached step fires no step-level hooks. Pipeline-level hooks on a dynamic run still fire even when every step was cached.
- **Hook failures are swallowed.** When a hook raises, the run or step is not aborted. The exception is captured into the `HookInvocation` record with `status=FAILED` and execution continues.
- **Return values are discarded** by default. Set `ZENML_TRACK_LIFECYCLE_HOOK_OUTPUTS=true` in the execution environment to materialize lifecycle hook return values as output artifacts of the invocation.

## `on_init` and `on_cleanup` are different

`on_init` and `on_cleanup` are pipeline setup and teardown hooks. They initialize and tear down shared run state once per execution environment, not per run or step, so they are **not** recorded as `HookInvocation` records.

- **Deployments:** `on_init` runs once per replica at startup, `on_cleanup` once at shutdown. Individual invocations reuse the initialized state. Lifecycle hooks like `on_start`/`on_end` still fire on every invocation.
- **Regular runs:** `on_init` runs once per execution environment. For a dynamic pipeline, that is the orchestrator environment. For any step running outside the orchestration environment, it runs once ahead of that step body the first time its environment is used.

When `on_init` fails, the run still records `RUN_START`, `RUN_END`, and `RUN_FAILURE`, but no row for `on_init` itself. Find the root cause on `pipeline_run.exception_info`.

## Recording custom invocations with `run_hook`

Beyond the lifecycle hooks, record arbitrary invocations from inside a step or dynamic pipeline function. This is useful for instrumenting third-party callbacks such as the tool and model calls of an agent framework.

```python
from zenml import run_hook, step

def call_tool(name: str) -> str:
    return f"result of {name}"

@step
def agent_step():
    # Records one CUSTOM HookInvocation, returns the function's result.
    result = run_hook(call_tool, "search")
```

Pass `store_return=True` to materialize the return value as an output artifact. A single unannotated return becomes one artifact named `output`. An annotated tuple return unpacks into one artifact per element.

```python
result = run_hook(call_tool, "search", store_return=True)
```

## Querying hook invocations

```python
from zenml.client import Client
from zenml.enums import HookType

invocations = Client().list_hook_invocations(
    pipeline_run_id=run.id,
    hook_type=HookType.CUSTOM,
)
for invocation in invocations.items:
    print(invocation.name, invocation.status)
```

CLI equivalent: `zenml hook-invocation list`, filterable by run, step, hook type, and name.

**Docs:** https://docs.zenml.io/how-to/steps-pipelines/hooks
</content>
</invoke>
