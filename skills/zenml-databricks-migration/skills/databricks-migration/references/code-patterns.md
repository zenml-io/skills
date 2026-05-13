# Code Translation Patterns

Side-by-side Databricks Workflows -> ZenML translations for all major patterns. Each example shows the Databricks job JSON, the in-task code, and the ZenML equivalent with complete imports.

## Table of Contents

- [Linear Notebook Workflow](#linear-notebook-workflow)
- [Task Values Data Passing](#task-values-data-passing)
- [Branching with condition_task](#branching-with-condition_task)
- [For-Each Iteration](#for-each-iteration)
- [Mixed Task Types (Notebook + Wheel + SQL)](#mixed-task-types-notebook--wheel--sql)
- [Retries and Timeouts](#retries-and-timeouts)
- [Job Clusters and Per-Task Compute](#job-clusters-and-per-task-compute)
- [Run-Job Task (Pipeline Composition)](#run-job-task-pipeline-composition)
- [File Arrival Trigger](#file-arrival-trigger)
- [Notebooks with Spark and Widgets](#notebooks-with-spark-and-widgets)
- [Secrets and Connections](#secrets-and-connections)
- [YAML Configuration and Small CLI](#yaml-configuration-and-small-cli)
- [Databricks Feature Engineering / Unity Catalog Feature Lookup](#databricks-feature-engineering--unity-catalog-feature-lookup)
- [External Wheel and Workspace Dependencies](#external-wheel-and-workspace-dependencies)
- [Caching-Sensitive Steps](#caching-sensitive-steps)

---

## Linear Notebook Workflow

A simple ETL pipeline with job-level parameters passed to notebook tasks via widgets.

### Databricks Job JSON

```json
{
  "name": "etl_linear_with_params",
  "parameters": [
    { "name": "run_date", "default": "{{job.start_time.iso_date}}" },
    { "name": "input_table", "default": "raw.events" }
  ],
  "tasks": [
    {
      "task_key": "extract",
      "existing_cluster_id": "0201-123abc-cluster",
      "notebook_task": {
        "notebook_path": "/Repos/acme/etl/extract",
        "base_parameters": {
          "run_date": "{{job.parameters.run_date}}",
          "input_table": "{{job.parameters.input_table}}"
        }
      }
    },
    {
      "task_key": "transform",
      "depends_on": [{ "task_key": "extract" }],
      "existing_cluster_id": "0201-123abc-cluster",
      "notebook_task": { "notebook_path": "/Repos/acme/etl/transform" }
    },
    {
      "task_key": "load",
      "depends_on": [{ "task_key": "transform" }],
      "existing_cluster_id": "0201-123abc-cluster",
      "notebook_task": { "notebook_path": "/Repos/acme/etl/load" }
    }
  ]
}
```

### Databricks Notebook Code (extract notebook)

```python
# Databricks notebook source
dbutils.widgets.text("run_date", "")
dbutils.widgets.text("input_table", "")

run_date = dbutils.widgets.get("run_date")
input_table = dbutils.widgets.get("input_table")

df = spark.read.table(input_table).where(f"date = '{run_date}'")
df.createOrReplaceTempView("events_for_day")

print(f"Extracted rows for {run_date} from {input_table}: {df.count()}")
```

### ZenML

```python
from __future__ import annotations

from typing import Annotated

import pandas as pd
from zenml import pipeline, step


@step
def extract(run_date: str, input_table: str) -> Annotated[pd.DataFrame, "events_df"]:
    """Extract data for a given date.

    Migration note: Databricks notebook used spark.read.table() with implicit
    Spark session. Replace with your portable data access layer (warehouse
    connector, cloud storage client, etc.) for the target ZenML stack.
    """
    # TODO(migration): Replace with actual data source query
    return pd.DataFrame({"date": [run_date], "source": [input_table]})


@step
def transform(events_df: pd.DataFrame) -> Annotated[pd.DataFrame, "features_df"]:
    return events_df.assign(feature_x=1)


@step
def load(features_df: pd.DataFrame) -> None:
    # TODO(migration): Replace with warehouse write, feature store update, etc.
    print("Would load rows:", len(features_df))


@pipeline
def etl_linear_with_params(run_date: str, input_table: str = "raw.events") -> None:
    events = extract(run_date=run_date, input_table=input_table)
    features = transform(events)
    load(features)
```

**Key differences**:
- `{{job.start_time.iso_date}}` is a Databricks runtime substitution. In ZenML, compute the run date as a pipeline parameter (passed by scheduler/triggering system) or inside a step.
- `dbutils.widgets` calls become regular function parameters.
- `spark.read.table()` must be replaced with a portable data access pattern for the target stack.

---

## Task Values Data Passing

Databricks task values are the primary inter-task communication channel (similar to Airflow XCom but JSON-only and 48KiB limited).

### Databricks Job JSON

```json
{
  "name": "task_values_passing",
  "tasks": [
    {
      "task_key": "produce_message",
      "existing_cluster_id": "0201-123abc-cluster",
      "notebook_task": { "notebook_path": "/Repos/acme/taskvalues/produce" }
    },
    {
      "task_key": "consume_message",
      "depends_on": [{ "task_key": "produce_message" }],
      "existing_cluster_id": "0201-123abc-cluster",
      "notebook_task": {
        "notebook_path": "/Repos/acme/taskvalues/consume",
        "base_parameters": {
          "received_message": "{{tasks.produce_message.values.message}}"
        }
      }
    }
  ]
}
```

### Databricks Notebook Code (producer)

```python
# Databricks notebook source
message = "Hello from the first task"
dbutils.jobs.taskValues.set(key="message", value=message)
print(f"Produced message: {message}")
```

### ZenML

```python
from __future__ import annotations

from zenml import pipeline, step


@step
def produce_message() -> str:
    return "Hello from the first step"


@step
def consume_message(received_message: str) -> None:
    print(f"Received message: {received_message}")


@pipeline
def task_values_passing() -> None:
    msg = produce_message()
    consume_message(received_message=msg)
```

**Key differences**:
- `dbutils.jobs.taskValues.set()/get()` and `{{tasks.*.values.*}}` references are replaced by direct artifact passing through function calls.
- ZenML artifacts are typed and persisted in the artifact store (not ephemeral JSON blobs). This changes observability and retention characteristics.
- For truly small control signals, consider logging metadata instead of producing a durable artifact.

---

## Branching with condition_task

Databricks condition tasks evaluate operands (which can be dynamic references) and route execution to true/false branches.

### Databricks Job JSON

```json
{
  "name": "branching_with_condition",
  "tasks": [
    {
      "task_key": "check_budget",
      "condition_task": {
        "op": "LESS_THAN",
        "left": "{{job.parameters.repair_count}}",
        "right": "5"
      }
    },
    {
      "task_key": "normal_path",
      "depends_on": [{ "task_key": "check_budget", "outcome": "true" }],
      "existing_cluster_id": "0201-123abc-cluster",
      "notebook_task": { "notebook_path": "/Repos/acme/branch/normal" }
    },
    {
      "task_key": "fallback_path",
      "depends_on": [{ "task_key": "check_budget", "outcome": "false" }],
      "existing_cluster_id": "0201-123abc-cluster",
      "notebook_task": { "notebook_path": "/Repos/acme/branch/fallback" }
    }
  ]
}
```

### ZenML (parameter-based branching -- static pipeline)

If the condition depends only on a pipeline parameter (known at construction time):

```python
from __future__ import annotations

from zenml import pipeline, step


@step
def normal_path() -> str:
    return "normal path executed"


@step
def fallback_path() -> str:
    return "fallback path executed"


@pipeline
def branching_with_condition(repair_count: int = 0) -> None:
    if repair_count < 5:
        normal_path()
    else:
        fallback_path()
```

### ZenML (runtime branching -- dynamic pipeline)

If the condition depends on a value computed by an upstream step:

```python
from __future__ import annotations

from zenml import pipeline, step


@step
def compute_repair_count() -> int:
    # In a real migration, this reads from a database or API
    return 3


@step
def normal_path() -> None:
    print("Running normal path")


@step
def fallback_path() -> None:
    print("Running fallback path")


@pipeline(dynamic=True)
def branching_with_condition() -> None:
    count = compute_repair_count()
    if count.load() < 5:  # .load() fetches the runtime value
        normal_path()
    else:
        fallback_path()
```

**Key differences**:
- Databricks `condition_task` evaluates without running compute. ZenML needs either a pipeline parameter or a step to supply the value.
- Dynamic pipelines require orchestrator support -- flag this requirement in the migration report.
- Databricks condition operators (`LESS_THAN`, `GREATER_THAN`, `EQUAL_TO`) become Python comparison operators.

---

## For-Each Iteration

Databricks `for_each_task` iterates over a JSON array produced by an upstream task.

### Databricks Job JSON

```json
{
  "name": "for_each_iteration",
  "tasks": [
    {
      "task_key": "generate_items",
      "existing_cluster_id": "0201-123abc-cluster",
      "notebook_task": { "notebook_path": "/Repos/acme/foreach/generate_items" }
    },
    {
      "task_key": "process_items",
      "depends_on": [{ "task_key": "generate_items" }],
      "for_each_task": {
        "inputs": "{{tasks.generate_items.values.items}}",
        "concurrency": 10,
        "task": {
          "task_key": "process_item_iteration",
          "existing_cluster_id": "0201-123abc-cluster",
          "notebook_task": {
            "notebook_path": "/Repos/acme/foreach/process_one",
            "base_parameters": { "item": "{{input}}" }
          }
        }
      }
    }
  ]
}
```

### Databricks Notebook Code (generate_items)

```python
# Databricks notebook source
items = ["a", "b", "c"]
dbutils.jobs.taskValues.set(key="items", value=items)
```

### ZenML

```python
from __future__ import annotations

from typing import List

from zenml import pipeline, step


@step
def generate_items() -> List[str]:
    return ["a", "b", "c"]


@step
def process_one(item: str) -> None:
    print("processing", item)


@pipeline(dynamic=True)
def for_each_iteration() -> None:
    items = generate_items()
    # Fan-out: each item processed as a separate step instance
    process_one.map(item=items)
```

**Key differences**:
- Databricks `for_each_task` has explicit `concurrency` control. ZenML fan-out parallelism is orchestrator-dependent -- document the original concurrency setting in the migration report.
- Dynamic pipelines require `@pipeline(dynamic=True)` and orchestrator support.
- If cardinality is known at pipeline construction time (not from upstream step), you can use a static fan-out instead.

---

## Mixed Task Types (Notebook + Wheel + SQL)

A workflow combining different Databricks task types into a single ZenML pipeline.

### Databricks Job JSON

```json
{
  "name": "mixed_task_types",
  "tasks": [
    {
      "task_key": "prep_notebook",
      "existing_cluster_id": "0201-123abc-cluster",
      "notebook_task": { "notebook_path": "/Repos/acme/mixed/prep" }
    },
    {
      "task_key": "train_wheel",
      "depends_on": [{ "task_key": "prep_notebook" }],
      "existing_cluster_id": "0201-123abc-cluster",
      "python_wheel_task": {
        "package_name": "acme_model",
        "entry_point": "train",
        "named_parameters": { "epochs": "3" }
      },
      "libraries": [
        { "whl": "dbfs:/FileStore/wheels/acme_model-0.1.0-py3-none-any.whl" }
      ]
    },
    {
      "task_key": "publish_metrics_sql",
      "depends_on": [{ "task_key": "train_wheel" }],
      "sql_task": {
        "warehouse_id": "1a111111a1111aa1",
        "file": { "path": "/Repos/acme/mixed/publish_metrics.sql", "source": "WORKSPACE" },
        "parameters": { "run_id": "{{job.run_id}}" }
      }
    }
  ]
}
```

### ZenML

```python
from __future__ import annotations

from typing import Annotated, Dict

from zenml import pipeline, step
from zenml.config import DockerSettings


@step
def prep_data() -> Annotated[Dict[str, str], "prep_result"]:
    """Replaces notebook logic. Extract core functionality into plain Python."""
    # TODO(migration): Replace with actual data preparation logic from notebook
    return {"prepared": "true"}


@step(settings={"docker": DockerSettings(requirements=["acme-model==0.1.0"])})
def train_model(
    prep: Dict[str, str], epochs: int = 3
) -> Annotated[Dict[str, float], "metrics"]:
    """Replaces python_wheel_task.

    Migration note: Instead of running a wheel entry point indirectly via
    Databricks, import and call the training function directly. Dependencies
    are installed via DockerSettings.
    """
    # from acme_model.train import run_training
    # return run_training(epochs=epochs)
    _ = prep
    return {"accuracy": 0.9, "epochs": float(epochs)}


@step
def publish_metrics(metrics: Dict[str, float], run_id: str) -> None:
    """Replaces sql_task.

    Migration note: Databricks sql_task used warehouse_id and managed identity.
    This step uses explicit credentials from ZenML secrets. Ensure SQL connector
    is configured: `zenml secret create db-creds --key=token --value=...`
    """
    # import databricks.sql as dbsql
    # conn = dbsql.connect(server_hostname=..., http_path=..., access_token=...)
    # cursor = conn.cursor()
    # cursor.execute("INSERT INTO metrics ...", [metrics, run_id])
    print("Would publish metrics", metrics, "for run_id", run_id)


@pipeline
def mixed_task_types(epochs: int = 3, run_id: str = "UNKNOWN") -> None:
    prep = prep_data()
    metrics = train_model(prep, epochs=epochs)
    publish_metrics(metrics, run_id=run_id)
```

**Key differences**:
- Three different Databricks parameter channels (widgets, wheel args, SQL substitution) are replaced with one coherent Python parameter model, with business values supplied from ZenML YAML configs rather than a long CLI.
- `python_wheel_task` libraries become `DockerSettings(requirements=[...])`, `pyproject.toml` dependencies, or a private package artifact strategy when the original wheel is on DBFS/workspace storage.
- `sql_task` becomes explicit client code with credentials from ZenML secrets, not managed identity.
- `{{job.run_id}}` becomes a pipeline parameter (or use `get_step_context().pipeline_run.id` inside the step).

---

## Retries and Timeouts

### Databricks Job JSON

```json
{
  "name": "retries_and_timeouts",
  "tasks": [
    {
      "task_key": "flaky_step",
      "existing_cluster_id": "0201-123abc-cluster",
      "notebook_task": { "notebook_path": "/Repos/acme/ops/flaky" },
      "max_retries": 3,
      "min_retry_interval_millis": 60000,
      "timeout_seconds": 900,
      "retry_on_timeout": true
    }
  ]
}
```

### ZenML

```python
from __future__ import annotations

from zenml import pipeline, step
from zenml.config.retry_config import StepRetryConfig


@step(retry=StepRetryConfig(max_retries=3, delay=60, backoff=1))
def flaky_step() -> str:
    """Migration note: Databricks timeout_seconds=900 has no universal ZenML
    equivalent. For hard timeout enforcement:
    - Kubernetes orchestrator: set active_deadline_seconds in pod settings
    - Other orchestrators: use application-level timeout wrapper
    The retry_on_timeout=true behavior is preserved by StepRetryConfig (ZenML
    retries on any failure, including timeout).
    """
    # TODO(migration): Original notebook logic from /Repos/acme/ops/flaky
    return "ok"


@pipeline
def retries_and_timeouts() -> None:
    _ = flaky_step()
```

**Key differences**:
- `max_retries` maps directly to `StepRetryConfig(max_retries=N)`.
- `min_retry_interval_millis` maps to `StepRetryConfig(delay=N)` (convert ms to seconds).
- `timeout_seconds` has no universal ZenML equivalent. Use orchestrator-specific settings (e.g., Kubernetes `active_deadline_seconds`) or an application-level timeout wrapper.
- `retry_on_timeout` is implicit in ZenML (retries happen on any failure).

---

## Job Clusters and Per-Task Compute

### Databricks Job JSON

```json
{
  "name": "job_clusters_example",
  "job_clusters": [
    {
      "job_cluster_key": "cpu_cluster",
      "new_cluster": {
        "spark_version": "13.3.x-scala2.12",
        "node_type_id": "i3.xlarge",
        "num_workers": 2
      }
    },
    {
      "job_cluster_key": "gpu_cluster",
      "new_cluster": {
        "spark_version": "13.3.x-scala2.12",
        "node_type_id": "g5.2xlarge",
        "num_workers": 1
      }
    }
  ],
  "tasks": [
    {
      "task_key": "feature_eng",
      "job_cluster_key": "cpu_cluster",
      "notebook_task": { "notebook_path": "/Repos/acme/compute/feature_eng" }
    },
    {
      "task_key": "train",
      "depends_on": [{ "task_key": "feature_eng" }],
      "job_cluster_key": "gpu_cluster",
      "python_wheel_task": { "package_name": "acme_model", "entry_point": "train" }
    }
  ]
}
```

### ZenML

```python
from __future__ import annotations

from zenml import pipeline, step
from zenml.config import ResourceSettings


@step(settings={"resources": ResourceSettings(cpu_count=2, memory="8GiB")})
def feature_eng() -> str:
    """Migration note: Databricks ran this on cpu_cluster (i3.xlarge, 2 workers).
    ResourceSettings captures resource intent; actual enforcement depends on
    orchestrator/step operator.
    """
    return "features_ready"


@step(settings={"resources": ResourceSettings(cpu_count=4, gpu_count=1, memory="16GiB")})
def train(features_ready: str) -> None:
    """Migration note: Databricks ran this on gpu_cluster (g5.2xlarge).
    GPU scheduling is orchestrator-dependent. Verify the target stack
    supports GPU allocation.
    """
    _ = features_ready
    print("Training with GPU intent")


@pipeline
def job_clusters_example() -> None:
    features = feature_eng()
    train(features)
```

**Key differences**:
- Databricks explicitly manages cluster lifecycle (create/reuse/terminate). ZenML launches one container per step under the orchestrator's execution model.
- `ResourceSettings` captures resource intent (CPU/memory/GPU) but does not guarantee placement. Actual enforcement depends on the orchestrator.
- Shared job clusters in Databricks preserve warm Spark context across tasks. ZenML has no equivalent -- each step is isolated.

---

## Run-Job Task (Pipeline Composition)

### Databricks Job JSON

```json
{
  "name": "job_B_triggers_job_A",
  "tasks": [
    {
      "task_key": "run_job_A",
      "run_job_task": {
        "job_id": 12345,
        "job_parameters": { "mode": "full" }
      }
    }
  ]
}
```

### ZenML (pipeline composition -- same codebase)

```python
from __future__ import annotations

from zenml import pipeline, step


@step
def job_a_logic(mode: str) -> str:
    return f"job_a_ran_mode={mode}"


@pipeline
def pipeline_a(mode: str = "full") -> None:
    job_a_logic(mode=mode)


@pipeline
def pipeline_b_triggers_a() -> None:
    # Pipeline composition: call pipeline_a as a sub-pipeline
    pipeline_a(mode="full")
```

### ZenML (cross-workspace -- API trigger)

```python
from __future__ import annotations

from zenml import pipeline, step
from zenml.client import Client


@step
def trigger_external_pipeline(pipeline_name: str, mode: str) -> str:
    """Trigger another pipeline run via ZenML Client API.

    Migration note: Databricks run_job_task triggered job_id=12345 in the same
    workspace. For cross-workspace orchestration, use ZenML pipeline
    deployments/snapshots and trigger via API.
    """
    client = Client()
    # client.trigger_pipeline(pipeline_name, parameters={"mode": mode})
    return f"triggered {pipeline_name} with mode={mode}"


@pipeline
def pipeline_b_triggers_a() -> None:
    trigger_external_pipeline(pipeline_name="pipeline_a", mode="full")
```

**Key differences**:
- For code reuse within the same repo, pipeline composition is the clean ZenML equivalent.
- For cross-workspace orchestration (different permissions, repos), trigger a run via ZenML Client API or pipeline deployments.

---

## File Arrival Trigger

### Databricks Job JSON

```json
{
  "name": "file_arrival_ingestion",
  "trigger": {
    "file_arrival": {
      "url": "/Volumes/mycatalog/myschema/myvolume/inbox/",
      "min_time_between_triggers_seconds": 300,
      "wait_after_last_change_seconds": 60
    }
  },
  "tasks": [
    {
      "task_key": "ingest_new_files",
      "existing_cluster_id": "0201-123abc-cluster",
      "notebook_task": { "notebook_path": "/Repos/acme/ingest/autoloader" }
    }
  ]
}
```

### ZenML

```python
from __future__ import annotations

from zenml import pipeline, step


@step
def ingest_new_files(root_path: str) -> None:
    """Ingest files from a cloud storage location.

    Migration note: Databricks file_arrival trigger monitored a Unity Catalog
    volume for new files. ZenML OSS has no native file arrival trigger.
    Recommended architecture:
    1. Cloud storage event (S3 event, GCS notification, Azure Event Grid)
    2. Event -> message queue or webhook
    3. Webhook -> trigger ZenML pipeline run (via deployment/snapshot API)
    """
    # TODO(migration): Replace with cloud SDK listing + ingestion + idempotency
    print("Ingesting from:", root_path)


@pipeline
def file_arrival_ingestion(root_path: str) -> None:
    ingest_new_files(root_path=root_path)


# --- Triggering infrastructure (outside ZenML) ---
# Option 1: AWS Lambda + S3 event -> POST to ZenML pipeline deployment endpoint
# Option 2: GCS Pub/Sub -> Cloud Function -> ZenML Client API
# Option 3: Cron-based polling schedule as a simpler alternative
```

**Key differences**:
- This is an **absent** pattern. Databricks file arrival triggers are tightly coupled to Unity Catalog.
- ZenML OSS requires external eventing to replicate this pattern.
- Always flag file arrival triggers as HIGH-severity in the migration report.

---

## Notebooks with Spark and Widgets

Complex notebook that uses Spark session, widgets, temp views, and display -- the most common migration challenge.

### Databricks Notebook Code

```python
# Databricks notebook source
# Cell 1: Parameters
dbutils.widgets.text("date", "2024-01-01")
dbutils.widgets.dropdown("mode", "full", ["full", "incremental"])

date = dbutils.widgets.get("date")
mode = dbutils.widgets.get("mode")

# Cell 2: Read and filter
df = spark.read.table("catalog.schema.raw_events")
if mode == "incremental":
    df = df.where(f"date >= '{date}'")
df.createOrReplaceTempView("filtered_events")

# Cell 3: Transform (SQL magic)
# %sql
# CREATE OR REPLACE TEMP VIEW features AS
# SELECT user_id, COUNT(*) as event_count FROM filtered_events GROUP BY user_id

# Cell 4: Display and write
features_df = spark.table("features")
display(features_df.describe())
features_df.write.mode("overwrite").saveAsTable("catalog.schema.features")
```

### ZenML

```python
from __future__ import annotations

from typing import Annotated

import pandas as pd
from zenml import pipeline, step


@step
def read_and_filter(date: str, mode: str) -> Annotated[pd.DataFrame, "filtered_events"]:
    """Refactored from notebook cells 1-2.

    Migration note: Original notebook used spark.read.table() and temp views.
    Replace with your data access layer. If data scale requires Spark, consider:
    - Using ZenML's Databricks orchestrator (keeps Spark available)
    - Using a Spark step operator
    - Refactoring to pandas/polars if data fits in memory
    """
    # TODO(migration): Replace with actual data source query
    # Option A (Spark via step operator):
    #   from pyspark.sql import SparkSession
    #   spark = SparkSession.builder.getOrCreate()
    #   df = spark.read.table("catalog.schema.raw_events")
    # Option B (pandas via warehouse connector):
    #   import databricks.sql as dbsql
    #   conn = dbsql.connect(...)
    #   df = pd.read_sql("SELECT * FROM catalog.schema.raw_events", conn)
    df = pd.DataFrame({"user_id": [1, 2, 3], "date": [date] * 3})
    if mode == "incremental":
        df = df[df["date"] >= date]
    return df


@step
def compute_features(
    filtered_events: pd.DataFrame,
) -> Annotated[pd.DataFrame, "features"]:
    """Refactored from notebook cell 3 (%sql magic -> Python).

    Migration note: Original used %sql to create a temp view. In ZenML,
    transforms are explicit Python operations on artifact data.
    """
    return filtered_events.groupby("user_id").size().reset_index(name="event_count")


@step
def write_features(features: pd.DataFrame) -> None:
    """Refactored from notebook cell 4.

    Migration note: display() has no ZenML equivalent. Log summary stats as
    metadata instead. saveAsTable() requires warehouse/catalog access.
    """
    from zenml import log_metadata

    log_metadata(metadata={"row_count": len(features), "columns": list(features.columns)})
    # TODO(migration): Replace with actual write to feature store/warehouse
    print("Would write features:", len(features), "rows")


@pipeline
def notebook_pipeline(date: str = "2024-01-01", mode: str = "full") -> None:
    filtered = read_and_filter(date=date, mode=mode)
    features = compute_features(filtered)
    write_features(features)
```

**Key differences**:
- `dbutils.widgets` -> step function parameters
- `spark.read.table()` -> portable data access (connector, API, or Spark via step operator)
- `%sql` magic -> explicit Python transform
- `createOrReplaceTempView()` -> artifact passing (no shared Spark state)
- `display()` -> `log_metadata()` for observability
- `saveAsTable()` -> explicit write with credentials from ZenML secrets

---

## Secrets and Connections

### Databricks Notebook Code

```python
# Databricks notebook source
token = dbutils.secrets.get(scope="my-scope", key="api-token")
db_password = dbutils.secrets.get(scope="my-scope", key="db-password")

# Use secrets to connect to external service
import requests
resp = requests.get("https://api.example.com/data", headers={"Authorization": f"Bearer {token}"})
```

### ZenML

```python
from __future__ import annotations

from zenml import pipeline, step
from zenml.client import Client


@step
def fetch_external_data() -> dict:
    """Migration note: Databricks secrets (scope='my-scope') are migrated to
    ZenML secrets store. Create the secret:
      zenml secret create my-scope --key=api-token --value=...
      zenml secret create my-scope --key=db-password --value=...
    """
    client = Client()
    secret = client.get_secret("my-scope")
    token = secret.secret_values["api-token"]

    import requests

    resp = requests.get(
        "https://api.example.com/data",
        headers={"Authorization": f"Bearer {token}"},
    )
    return resp.json()


@pipeline
def secrets_example() -> None:
    fetch_external_data()
```

**Key differences**:
- `dbutils.secrets.get(scope, key)` -> `Client().get_secret(name).secret_values[key]`
- Databricks secret scopes map to ZenML secret names. Keys map 1:1.
- ZenML secrets can also be injected as environment variables using `DockerSettings(environment={"API_TOKEN": "{{my-scope.api-token}}"})`.

---

## YAML Configuration and Small CLI

Databricks job JSON often mixes DAG structure, business parameters, schedules, and per-task settings. In ZenML, keep Python focused on the DAG and use populated YAML configs for the knobs.

### Recommended pattern

```text
configs/dev.yaml   -> development table names, dates, resources, secrets refs
configs/prod.yaml  -> production table names, schedules, resource sizes
run.py             -> chooses --config and optional operational flags only
```

### Migration rule

- Move job parameters, widget defaults, table names, feature table names, hyperparameters, and environment-specific settings into `configs/dev.yaml` and `configs/prod.yaml`.
- Keep `argparse` small: `--config`, `--no-cache`, `--dry-run`, or similar operational switches.
- Avoid creating one CLI argument per Databricks parameter; that recreates a fragile job UI instead of using ZenML's config system.

---

## Databricks Feature Engineering / Unity Catalog Feature Lookup

Do not treat Feature Engineering Client or Unity Catalog feature lookup code as ordinary SQL unless the user explicitly accepts that semantic change.

### Databricks pattern

```python
from databricks.feature_engineering import FeatureEngineeringClient, FeatureLookup

fe = FeatureEngineeringClient()
training_set = fe.create_training_set(
    df=labels_df,
    feature_lookups=[
        FeatureLookup(
            table_name="catalog.schema.customer_features",
            lookup_key="customer_id",
            timestamp_lookup_key="event_ts",
        )
    ],
    label="churned",
)
training_df = training_set.load_df()
```

### Migration guidance

- If the target execution stack remains Databricks, preserve this as Databricks-native feature access inside a ZenML step and make credentials/dependencies explicit.
- If moving away from Databricks, flag for design review before rewriting. Preserve the contract: lookup keys, timestamp keys, point-in-time correctness, online/offline behavior, table ownership, and Unity Catalog permissions.
- Only use a Databricks SQL connector when the feature read is truly a simple table read and point-in-time feature semantics are not required.

---

## External Wheel and Workspace Dependencies

Databricks can install wheels from DBFS/workspace paths that the ZenML container builder cannot see.

### Databricks job fragment

```json
{
  "python_wheel_task": {"package_name": "acme_model", "entry_point": "train"},
  "libraries": [{"whl": "dbfs:/FileStore/wheels/acme_model-0.1.0-py3-none-any.whl"}]
}
```

### Migration guidance

- If source is available, make it an importable Python package and call the underlying function from the ZenML step.
- If the wheel is external and versioned, publish it to a private package index or make it available as a Docker build artifact/direct URL.
- If it exists only in DBFS/workspace and cannot be resolved, mark it as a blocker with the exact path. Do not emit a vague dependency TODO.

---

## Caching-Sensitive Steps

ZenML caching can skip steps whose declared inputs and code have not changed. Disable caching or document the decision when a migrated task depends on hidden inputs.

Cache-sensitive examples:
- `{{job.run_id}}`, `{{job.start_time.*}}`, `datetime.now()`, random seeds, or latest-partition logic
- reads from mutable Delta/Unity Catalog tables, feature tables, APIs, or cloud prefixes without an explicit version/partition/timestamp input
- writes, model registration, feature publishing, metric logging, notifications, and audit side effects

Migration rule: make the hidden value an explicit parameter if it should affect caching; otherwise set caching off for the side-effecting step.
