# Code Translation Patterns

Side-by-side Airflow → ZenML translations for all major patterns. Each example shows complete, runnable code with imports.

## Table of Contents

- [Simple Linear DAG](#simple-linear-dag)
- [TaskFlow API (XComArgs)](#taskflow-api-xcomargs)
- [XCom Data Passing (Classic Style)](#xcom-data-passing-classic-style)
- [Branching Logic](#branching-logic)
- [Dynamic Task Mapping](#dynamic-task-mapping)
- [Retry and Error Handling](#retry-and-error-handling)
- [Sensors](#sensors)
- [TaskGroups](#taskgroups)
- [Runtime Parameters](#runtime-parameters)
- [KubernetesPodOperator](#kubernetespodoperator)
- [Callbacks and Notifications](#callbacks-and-notifications)

---

## Simple Linear DAG

### Airflow

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator


def extract_numbers() -> list[int]:
    return [1, 2, 3, 4]


def sum_numbers(numbers: list[int]) -> int:
    return sum(numbers)


def print_result(total: int) -> None:
    print(f"Total = {total}")


with DAG(
    dag_id="linear_example",
    start_date=datetime(2024, 1, 1),
    schedule="@daily",
    catchup=False,
    default_args={"retries": 2, "retry_delay": timedelta(minutes=5)},
) as dag:
    extract = PythonOperator(task_id="extract", python_callable=extract_numbers)
    sum_ = PythonOperator(
        task_id="sum",
        python_callable=sum_numbers,
        op_kwargs={"numbers": "{{ ti.xcom_pull(task_ids='extract') }}"},
    )
    show = PythonOperator(
        task_id="show",
        python_callable=print_result,
        op_kwargs={"total": "{{ ti.xcom_pull(task_ids='sum') }}"},
    )
    extract >> sum_ >> show
```

### ZenML

```python
from typing import List
from zenml import pipeline, step
from zenml.config.retry_config import StepRetryConfig

RETRY = StepRetryConfig(max_retries=2, delay=300)


@step(retry=RETRY)
def extract_numbers() -> List[int]:
    return [1, 2, 3, 4]


@step(retry=RETRY)
def sum_numbers(numbers: List[int]) -> int:
    return sum(numbers)


@step(retry=RETRY)
def print_result(total: int) -> None:
    print(f"Total = {total}")


@pipeline
def linear_pipeline() -> None:
    numbers = extract_numbers()
    total = sum_numbers(numbers)
    print_result(total)
```

**Key differences**: Airflow's explicit `xcom_pull` in templates becomes plain function-call wiring. Returned values are artifacts persisted in the artifact store (not ephemeral XCom entries). `default_args` retries become `StepRetryConfig` applied to each step.

---

## TaskFlow API (XComArgs)

### Airflow

```python
from airflow.decorators import dag, task
from airflow.utils.dates import days_ago


@dag(dag_id="taskflow_example", start_date=days_ago(1), schedule=None, catchup=False)
def taskflow_example():
    @task
    def extract() -> list[int]:
        return [1, 2, 3]

    @task
    def transform(xs: list[int]) -> list[int]:
        return [x * 10 for x in xs]

    @task
    def load(xs: list[int]) -> None:
        print(xs)

    load(transform(extract()))


dag = taskflow_example()
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
def taskflow_style_pipeline() -> None:
    load(transform(extract()))
```

**Key difference**: This is the most direct translation — both systems model a dataflow graph by composing decorated functions. The persistence semantics differ (XCom vs artifact store), but the code structure is nearly identical.

---

## XCom Data Passing (Classic Style)

### Airflow

```python
from datetime import datetime
from airflow import DAG
from airflow.operators.python import PythonOperator


def produce() -> dict:
    return {"threshold": 0.8, "model_name": "demo"}


def consume(config: dict) -> None:
    print(f"Config = {config}")


with DAG(dag_id="xcom_example", start_date=datetime(2024, 1, 1), schedule=None, catchup=False) as dag:
    t1 = PythonOperator(task_id="produce", python_callable=produce)
    t2 = PythonOperator(
        task_id="consume",
        python_callable=consume,
        op_kwargs={"config": "{{ ti.xcom_pull(task_ids='produce') }}"},
    )
    t1 >> t2
```

### ZenML

```python
from typing import Any, Dict
from zenml import pipeline, step


@step
def produce() -> Dict[str, Any]:
    return {"threshold": 0.8, "model_name": "demo"}


@step
def consume(config: Dict[str, Any]) -> None:
    print(f"Config = {config}")


@pipeline
def xcom_equivalent_pipeline() -> None:
    cfg = produce()
    consume(cfg)
```

**Migration warning**: If the Airflow DAG uses XCom as a "side channel" for control-plane decisions (branching, triggering), flag it — ZenML cannot reproduce Airflow's runtime scheduler decisions based on XCom values.

---

## Branching Logic

### Airflow

```python
from datetime import datetime
from airflow import DAG
from airflow.operators.empty import EmptyOperator
from airflow.operators.python import BranchPythonOperator, PythonOperator


def choose_branch(**context) -> str:
    do_train = context["params"].get("do_train", True)
    return "train" if do_train else "skip_train"


with DAG(
    dag_id="branch_example",
    start_date=datetime(2024, 1, 1),
    schedule=None,
    catchup=False,
    params={"do_train": True},
) as dag:
    start = EmptyOperator(task_id="start")
    branch = BranchPythonOperator(task_id="branch", python_callable=choose_branch)
    train = PythonOperator(task_id="train", python_callable=lambda: print("Training..."))
    skip_train = PythonOperator(task_id="skip_train", python_callable=lambda: print("Skipping..."))
    join = EmptyOperator(task_id="join", trigger_rule="none_failed_min_one_success")
    start >> branch >> [train, skip_train] >> join
```

### ZenML (parameter-driven conditional)

```python
from zenml import pipeline, step


@step
def train_model() -> None:
    print("Training...")


@step
def skip_training() -> None:
    print("Skipping...")


@pipeline
def branch_pipeline(do_train: bool = True) -> None:
    # This condition evaluates at pipeline construction time,
    # so it must depend on pipeline parameters, not step outputs.
    if do_train:
        train_model()
    else:
        skip_training()
```

**Migration warning — HIGH severity**: Airflow branching is a runtime scheduling decision that produces "skipped" tasks and often relies on trigger rules (like `none_failed_min_one_success`) to join branches. ZenML can only branch on **pipeline parameters** (values known at construction time).

**If the Airflow branch decision depends on an upstream task's output** (e.g., branching based on data quality score from a previous step), this pattern **cannot be directly migrated**. Redesign options:
1. Run both branches but have each check a condition internally and no-op when irrelevant
2. Split into separate pipelines triggered by different conditions
3. Use a dynamic pipeline (`@pipeline(dynamic=True)`) where you `.load()` the upstream output and conditionally invoke steps

---

## Dynamic Task Mapping

### Airflow (2.3+)

```python
from airflow.decorators import dag, task
from airflow.utils.dates import days_ago


@dag(dag_id="dynamic_mapping", start_date=days_ago(1), schedule=None, catchup=False)
def dynamic_mapping():
    @task
    def list_items() -> list[int]:
        return [1, 2, 3, 4]

    @task
    def process(item: int) -> int:
        return item * 2

    @task
    def aggregate(results: list[int]) -> None:
        print(f"results={results}")

    items = list_items()
    mapped = process.expand(item=items)
    aggregate(mapped)


dag = dynamic_mapping()
```

### ZenML — Option A: Static fan-out (cardinality known at construction time)

Use when the list is a pipeline parameter, not an upstream step output:

```python
from typing import List
from zenml import pipeline, step


@step
def process(item: int) -> int:
    return item * 2


@step
def aggregate(results: List[int]) -> None:
    print(f"results={results}")


@pipeline(enable_cache=False)
def mapped_pipeline(items: List[int] = [1, 2, 3, 4]) -> None:
    outputs = []
    for i, item in enumerate(items):
        out = process(item, id=f"process_{i}")
        outputs.append(out)
    aggregate(outputs)
```

### ZenML — Option B: Dynamic pipeline (runtime-determined cardinality)

Use when the list comes from an upstream step:

```python
from typing import List
from zenml import pipeline, step


@step
def list_items() -> List[int]:
    return [1, 2, 3, 4]


@step
def process(item: int) -> int:
    return item * 2


@step
def aggregate(results: List[int]) -> None:
    print(f"results={results}")


@pipeline(dynamic=True)
def dynamic_mapped_pipeline() -> None:
    items = list_items()
    results = process.map(items)
    aggregate(results)
```

### ZenML — Option C: Multi-run pattern (for heavy fan-out with independent failure domains)

When each item is a substantial workload that should be independently retriable and observable:

```python
from typing import Dict, List
from uuid import UUID
import time
from zenml import pipeline, step
from zenml.client import Client


@step
def discover_work_items() -> List[str]:
    return ["chunk_1", "chunk_2", "chunk_3"]


@step
def trigger_worker_runs(chunks: List[str]) -> List[UUID]:
    client = Client()
    run_ids = []
    for c in chunks:
        run = client.trigger_pipeline(
            pipeline_name="chunk_worker_pipeline",
            run_configuration={"parameters": {"chunk_id": c}},
        )
        run_ids.append(run.id)

    completed = set()
    while len(completed) < len(run_ids):
        for rid in run_ids:
            if rid not in completed:
                r = client.get_pipeline_run(rid)
                if r.status.is_finished:
                    completed.add(rid)
        time.sleep(10)
    return run_ids


@step
def aggregate_results(run_ids: List[UUID]) -> Dict[str, object]:
    client = Client()
    out = {}
    for rid in run_ids:
        run = client.get_pipeline_run(rid)
        if run.status.value != "failed" and "process_chunk" in run.steps:
            out[str(rid)] = run.steps["process_chunk"].output.load()
    return out


@pipeline(enable_cache=False)
def orchestrator_pipeline() -> None:
    chunks = discover_work_items()
    run_ids = trigger_worker_runs(chunks)
    aggregate_results(run_ids)
```

**Migration warning — HIGH severity**: Airflow's `expand()` creates parallel task instances within a single DAG run, each with independent retry and observability. ZenML's options are:
- **Option A** changes the semantics (items must be known at construction time)
- **Option B** uses dynamic pipelines (experimental, limited orchestrator support)
- **Option C** changes the architecture (multiple pipeline runs instead of one)

Always flag `expand()` patterns and let the user choose the appropriate option.

---

## Retry and Error Handling

### Airflow

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator


def notify_failure(context) -> None:
    print(f"Task failed: {context['task_instance'].task_id}")


with DAG(
    dag_id="retry_example",
    start_date=datetime(2024, 1, 1),
    schedule=None,
    catchup=False,
    default_args={
        "retries": 3,
        "retry_delay": timedelta(seconds=10),
        "retry_exponential_backoff": True,
        "max_retry_delay": timedelta(minutes=5),
        "on_failure_callback": notify_failure,
    },
) as dag:
    t = PythonOperator(task_id="flaky", python_callable=lambda: (_ for _ in ()).throw(RuntimeError("transient")))
```

### ZenML

```python
from zenml import pipeline, step
from zenml.config.retry_config import StepRetryConfig
from zenml.hooks import alerter_failure_hook


@step(
    retry=StepRetryConfig(max_retries=3, delay=10, backoff=2),
    on_failure=alerter_failure_hook,
)
def flaky() -> None:
    raise RuntimeError("transient failure")


@pipeline
def retry_pipeline() -> None:
    flaky()
```

**Translation notes**:
- `retry_exponential_backoff=True` (boolean) → `backoff=2` (numeric factor). Choose an explicit factor.
- `max_retry_delay` has no ZenML equivalent. Note in the migration report if the original DAG relied on it.
- `alerter_failure_hook` posts to the stack's configured alerter (Slack/Discord). For custom logic, write a plain function: `def on_fail(exc: BaseException) -> None: ...`
- Orchestrators may add their own retry logic (e.g., Kubernetes pod restart). Document this to avoid over-retrying with non-idempotent operations.

---

## Sensors

### Airflow

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.sensors.time_delta import TimeDeltaSensor
from airflow.operators.python import PythonOperator


with DAG(dag_id="sensor_example", start_date=datetime(2024, 1, 1), schedule=None, catchup=False) as dag:
    wait = TimeDeltaSensor(
        task_id="wait_5_minutes",
        delta=timedelta(minutes=5),
        mode="reschedule",
        timeout=60 * 60,
    )
    t = PythonOperator(task_id="downstream", python_callable=lambda: print("Sensor satisfied"))
    wait >> t
```

### ZenML (polling step approximation)

```python
import time
from datetime import datetime, timedelta
from zenml import pipeline, step


@step
def wait_until(target_ts: float, timeout_seconds: int = 3600, poll_seconds: int = 30) -> None:
    """Polling step that approximates an Airflow sensor.

    Migration note: Unlike Airflow sensors with mode='reschedule', this step
    holds a compute slot (container/process) for its entire duration. For heavy
    sensor usage, consider redesigning with orchestrator scheduling or external
    event triggers instead.
    """
    start = time.time()
    while True:
        if time.time() >= target_ts:
            return
        if time.time() - start > timeout_seconds:
            raise TimeoutError("wait_until timed out")
        time.sleep(poll_seconds)


@step
def downstream() -> None:
    print("Condition satisfied")


@pipeline
def sensor_equivalent_pipeline(wait_minutes: int = 5) -> None:
    target = (datetime.utcnow() + timedelta(minutes=wait_minutes)).timestamp()
    wait_until(target_ts=target, timeout_seconds=3600, poll_seconds=30)
    downstream()
```

**Migration warning — HIGH severity**: Airflow sensors have dedicated scheduling semantics (`poke` vs `reschedule` mode) and can be fully deferrable via the triggerer process. A ZenML polling step consumes a compute slot for its entire run. For DAGs with heavy sensor usage, recommend:
1. Redesign with orchestrator scheduling (run the pipeline on a schedule instead of waiting)
2. Use external event triggers (webhook → pipeline trigger)
3. If polling is unavoidable, keep poll intervals reasonable and set aggressive timeouts

---

## TaskGroups

### Airflow

```python
from datetime import datetime
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.utils.task_group import TaskGroup


with DAG(dag_id="taskgroup_example", start_date=datetime(2024, 1, 1), schedule=None, catchup=False) as dag:
    with TaskGroup(group_id="preprocessing") as preprocessing:
        t1 = PythonOperator(task_id="clean", python_callable=lambda: print("Cleaning"))
        t2 = PythonOperator(task_id="validate", python_callable=lambda: print("Validating"))
        t1 >> t2

    t3 = PythonOperator(task_id="train", python_callable=lambda: print("Training"))
    preprocessing >> t3
```

### ZenML

```python
from zenml import pipeline, step


@step
def clean() -> str:
    print("Cleaning")
    return "cleaned"


@step
def validate(data: str) -> str:
    print("Validating")
    return "validated"


@step
def train(data: str) -> None:
    print("Training")


def preprocessing() -> str:
    """Composition function replacing Airflow TaskGroup.

    This provides code-level grouping and reusability. No UI grouping
    equivalent exists in ZenML.
    """
    cleaned = clean()
    validated = validate(cleaned)
    return validated


@pipeline
def taskgroup_equivalent_pipeline() -> None:
    preprocessed = preprocessing()
    train(preprocessed)
```

**Translation note**: Airflow TaskGroups are a UI concept (grouping in the graph view) plus default-arg scoping. ZenML replaces this with plain Python functions that wire steps together. The function approach is more flexible (supports parameters, conditionals) but loses the visual grouping in the dashboard.

---

## Runtime Parameters

### Airflow

```python
from datetime import datetime
from airflow import DAG
from airflow.operators.python import PythonOperator


def use_config(**context) -> None:
    params = context["params"]
    dagrun_conf = (context.get("dag_run") and context["dag_run"].conf) or {}
    print(f"params={params}, dag_run.conf={dagrun_conf}")


with DAG(
    dag_id="params_example",
    start_date=datetime(2024, 1, 1),
    schedule=None,
    catchup=False,
    params={"threshold": 0.8},
) as dag:
    t = PythonOperator(task_id="use_config", python_callable=use_config)
```

### ZenML

```python
from zenml import pipeline, step


@step
def use_config(threshold: float) -> None:
    print(f"threshold={threshold}")


@pipeline
def params_pipeline(threshold: float = 0.8) -> None:
    use_config(threshold)


if __name__ == "__main__":
    # Override at runtime (equivalent to dag_run.conf):
    params_pipeline(threshold=0.9)
```

**Translation note**: Airflow has a complex param/conf override system (`dag_run_conf_overrides_params`). ZenML's equivalent is "pipeline arguments at invocation time" and/or YAML config. The precedence is: Runtime Python > Step-level YAML > Pipeline-level YAML > Defaults.

---

## KubernetesPodOperator

### Airflow

```python
from datetime import datetime
from airflow import DAG
from airflow.providers.cncf.kubernetes.operators.pod import KubernetesPodOperator


with DAG(dag_id="k8s_example", start_date=datetime(2024, 1, 1), schedule=None, catchup=False) as dag:
    run_job = KubernetesPodOperator(
        task_id="run_container",
        name="run-container",
        namespace="default",
        image="python:3.11-slim",
        cmds=["python", "-c"],
        arguments=["print('hello from k8s')"],
        get_logs=True,
        is_delete_operator_pod=True,
    )
```

### ZenML

```python
from zenml import pipeline, step
from zenml.config import DockerSettings


@step(step_operator="kubernetes")
def run_containerised_work() -> None:
    """Migration note: Airflow's KubernetesPodOperator ran an arbitrary container.
    ZenML's Kubernetes step operator runs your step code in a pod using ZenML's
    containerization. If the original task ran a non-Python container or arbitrary
    shell commands, you may need to adapt the logic to Python or use subprocess.
    """
    print("hello from k8s")


@pipeline(settings={"docker": DockerSettings(requirements=["zenml"])})
def k8s_pipeline() -> None:
    run_containerised_work()
```

**Translation note**: Airflow's `KubernetesPodOperator` executes arbitrary container images with arbitrary commands. ZenML's Kubernetes step operator executes your step code inside ZenML-built container images. Migration is straightforward when the original task runs Python code. For tasks that run non-Python binaries or specific Docker images, you may need:
- `subprocess.run()` inside a step (for shell commands)
- `DockerSettings(parent_image="...")` (for specific base images)
- A complete redesign if the task is fundamentally "run this external container" with no Python equivalent

---

## Callbacks and Notifications

### Airflow

```python
from datetime import datetime
from airflow import DAG
from airflow.operators.python import PythonOperator


def on_success(context) -> None:
    print(f"SUCCESS: {context['task_instance'].task_id}")


def on_failure(context) -> None:
    print(f"FAILURE: {context['task_instance'].task_id}")


with DAG(dag_id="callbacks_example", start_date=datetime(2024, 1, 1), schedule=None, catchup=False) as dag:
    t = PythonOperator(
        task_id="work",
        python_callable=lambda: print("doing work"),
        on_success_callback=on_success,
        on_failure_callback=on_failure,
    )
```

### ZenML

```python
from zenml import pipeline, step


def on_success() -> None:
    # ZenML success hooks take no arguments (unlike Airflow's context dict)
    print("SUCCESS")


def on_failure(exc: BaseException) -> None:
    # ZenML failure hooks receive the exception
    print(f"FAILURE: {exc!r}")


@step(on_success=on_success, on_failure=on_failure)
def work() -> None:
    print("doing work")


@pipeline
def callbacks_pipeline() -> None:
    work()
```

**For chat notifications**, use ZenML's standard alerter hooks (posts to the alerter configured in the active stack):

```python
from zenml.hooks import alerter_failure_hook, alerter_success_hook

@step(on_failure=alerter_failure_hook, on_success=alerter_success_hook)
def work() -> None:
    print("doing work")
```

**Translation note**: Airflow callbacks receive a rich context dict with `task_instance`, `dag_run`, `execution_date`, etc. ZenML hooks are simpler — success hooks take no args, failure hooks receive the exception. If the original callback accessed Airflow-specific context (like `execution_date` or `dag_run.conf`), the logic needs adaptation.
