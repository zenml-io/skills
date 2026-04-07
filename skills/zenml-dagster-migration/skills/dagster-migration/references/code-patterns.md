# Code Translation Patterns

Side-by-side Dagster -> ZenML translations for the major migration patterns. Each example shows the Dagster source, the ZenML translation, and the key semantic differences that still matter after the code is converted.

## Table of Contents

- [Simple asset graph -> single pipeline](#simple-asset-graph---single-pipeline)
- [Op / graph / job -> pipeline](#op--graph--job---pipeline)
- [IO manager -> explicit source and sink steps](#io-manager---explicit-source-and-sink-steps)
- [ConfigurableResource -> secrets, connectors, and helper objects](#configurableresource---secrets-connectors-and-helper-objects)
- [Config / RunConfig -> typed parameters and YAML config](#config--runconfig---typed-parameters-and-yaml-config)
- [Partitioned asset -> parameterized pipeline](#partitioned-asset---parameterized-pipeline)
- [Schedule -> `Schedule(...)`](#schedule---schedule)
- [Sensor -> external trigger pattern](#sensor---external-trigger-pattern)
- [Asset check -> validation step](#asset-check---validation-step)
- [multi_asset -> multi-output step](#multi_asset---multi-output-step)
- [graph_asset -> helper steps inside a pipeline](#graph_asset---helper-steps-inside-a-pipeline)
- [DynamicOutput fan-out -> dynamic pipeline or redesign](#dynamicoutput-fan-out---dynamic-pipeline-or-redesign)

---

## Simple asset graph -> single pipeline

This is the cleanest asset-style migration: one small Dagster asset graph that already behaves like one execution unit.

### Dagster

```python
import dagster as dg
import pandas as pd


@dg.asset
def raw_orders() -> pd.DataFrame:
    return pd.DataFrame({"order_id": [1, 2], "amount": [10.0, 20.0]})


@dg.asset
def cleaned_orders(raw_orders: pd.DataFrame) -> pd.DataFrame:
    return raw_orders.assign(amount_usd=raw_orders["amount"].fillna(0.0))


@dg.asset
def order_summary(cleaned_orders: pd.DataFrame) -> dict[str, float]:
    return {"total": float(cleaned_orders["amount_usd"].sum())}
```

### ZenML

```python
from __future__ import annotations

from typing import Annotated

import pandas as pd
from zenml import pipeline, step


@step
def raw_orders() -> Annotated[pd.DataFrame, "raw_orders"]:
    return pd.DataFrame({"order_id": [1, 2], "amount": [10.0, 20.0]})


@step
def cleaned_orders(raw_orders: pd.DataFrame) -> Annotated[pd.DataFrame, "cleaned_orders"]:
    return raw_orders.assign(amount_usd=raw_orders["amount"].fillna(0.0))


@step
def order_summary(cleaned_orders: pd.DataFrame) -> dict[str, float]:
    return {"total": float(cleaned_orders["amount_usd"].sum())}


@pipeline
def orders_pipeline() -> None:
    raw = raw_orders()
    cleaned = cleaned_orders(raw)
    order_summary(cleaned)
```

**Key differences**:
- The Dagster assets were first-class graph nodes. In ZenML, these become step outputs produced inside one pipeline run.
- If the original team materialized `cleaned_orders` independently, this translation is only approximate; split the graph into multiple pipelines instead.

---

## Op / graph / job -> pipeline

Op/job-centric Dagster code usually maps more directly because the source model already centers execution units.

### Dagster

```python
import dagster as dg


@dg.op
def extract() -> list[int]:
    return [1, 2, 3]


@dg.op
def transform(numbers: list[int]) -> list[int]:
    return [n * 10 for n in numbers]


@dg.op
def load(numbers: list[int]) -> None:
    print(numbers)


@dg.graph
def etl_graph():
    load(transform(extract()))


etl_job = etl_graph.to_job(name="etl_job")
```

### ZenML

```python
from zenml import pipeline, step


@step
def extract() -> list[int]:
    return [1, 2, 3]


@step
def transform(numbers: list[int]) -> list[int]:
    return [n * 10 for n in numbers]


@step
def load(numbers: list[int]) -> None:
    print(numbers)


@pipeline
def etl_pipeline() -> None:
    load(transform(extract()))
```

**Key differences**:
- This is close to a direct translation.
- Dagster graph/job objects disappear; ZenML uses plain Python composition inside the pipeline function.

---

## IO manager -> explicit source and sink steps

This is the pattern where migrations often go wrong. If the IO manager contains data-access behavior, move that behavior into steps.

### Dagster

```python
import dagster as dg
import pandas as pd


class WarehouseIOManager(dg.IOManager):
    def handle_output(self, context, obj: pd.DataFrame) -> None:
        obj.to_parquet(f"/warehouse/{context.asset_key.path[-1]}.parquet")

    def load_input(self, context) -> pd.DataFrame:
        upstream_name = context.upstream_output.asset_key.path[-1]
        return pd.read_parquet(f"/warehouse/{upstream_name}.parquet")


@dg.asset(io_manager_key="warehouse_io")
def orders() -> pd.DataFrame:
    return pd.DataFrame({"order_id": [1, 2]})


@dg.asset(io_manager_key="warehouse_io")
def order_count(orders: pd.DataFrame) -> int:
    return len(orders)
```

### ZenML

```python
from __future__ import annotations

import pandas as pd
from zenml import pipeline, step


@step
def load_orders_from_warehouse() -> pd.DataFrame:
    # Migration note: the original Dagster IOManager controlled upstream loading.
    # In ZenML, that data-access behavior is now explicit.
    return pd.read_parquet("/warehouse/orders.parquet")


@step
def compute_order_count(orders: pd.DataFrame) -> int:
    return len(orders)


@step
def persist_order_count(order_count: int) -> None:
    with open("/warehouse/order_count.txt", "w") as f:
        f.write(str(order_count))


@pipeline
def orders_pipeline() -> None:
    orders = load_orders_from_warehouse()
    count = compute_order_count(orders)
    persist_order_count(count)
```

**Key differences**:
- The load/write behavior is now explicit step logic.
- Use a materializer only for serialization concerns, not as a hiding place for business-specific warehouse conventions.

**Migration warning**:
- If the IO manager encoded lazy loading, environment routing, or partition-aware path resolution, flag it for review.

---

## ConfigurableResource -> secrets, connectors, and helper objects

### Dagster

```python
import dagster as dg


class ApiResource(dg.ConfigurableResource):
    base_url: str
    api_key: str

    def fetch(self, endpoint: str) -> dict:
        return {"endpoint": endpoint, "base_url": self.base_url}


@dg.asset
def customer_snapshot(api: ApiResource) -> dict:
    return api.fetch("/customers")
```

### ZenML

```python
from zenml import step


class ApiClient:
    def __init__(self, base_url: str, api_key: str) -> None:
        self.base_url = base_url
        self.api_key = api_key

    def fetch(self, endpoint: str) -> dict:
        return {"endpoint": endpoint, "base_url": self.base_url}


@step
def customer_snapshot(base_url: str, api_key: str) -> dict:
    client = ApiClient(base_url=base_url, api_key=api_key)
    return client.fetch("/customers")
```

**Key differences**:
- In Dagster, the framework injects the resource.
- In ZenML, make the dependency explicit and decide separately where the values come from: pipeline params, YAML config, secrets, or service connectors.

---

## Config / RunConfig -> typed parameters and YAML config

### Dagster

```python
import dagster as dg


class TrainConfig(dg.Config):
    learning_rate: float
    epochs: int


@dg.asset
def train_model(config: TrainConfig) -> dict:
    return {"lr": config.learning_rate, "epochs": config.epochs}
```

### ZenML

```python
from zenml import pipeline, step


@step
def train_model(learning_rate: float, epochs: int) -> dict:
    return {"lr": learning_rate, "epochs": epochs}


@pipeline
def training_pipeline(learning_rate: float, epochs: int) -> None:
    train_model(learning_rate=learning_rate, epochs=epochs)
```

**Key differences**:
- Dagster config object becomes explicit typed parameters.
- Environment-specific values can live in `configs/dev.yaml` and `configs/prod.yaml`.

---

## Partitioned asset -> parameterized pipeline

### Dagster

```python
import dagster as dg


daily = dg.DailyPartitionsDefinition(start_date="2024-01-01")


@dg.asset(partitions_def=daily)
def sales_for_day(context: dg.AssetExecutionContext) -> str:
    return context.partition_key
```

### ZenML

```python
from zenml import pipeline, step


@step
def sales_for_day(partition_key: str) -> str:
    # Migration note: this preserves the partition label, not Dagster's
    # partition engine, mappings, or backfill semantics.
    return partition_key


@pipeline
def sales_pipeline(partition_key: str) -> None:
    sales_for_day(partition_key=partition_key)
```

**Key differences**:
- This preserves the partition key, not the full partition orchestration model.
- Non-trivial partition mappings or coordinated backfills should be flagged, not silently collapsed into a parameter.

---

## Schedule -> `Schedule(...)`

### Dagster

```python
import dagster as dg


@dg.schedule(cron_schedule="0 2 * * *", job=my_job)
def daily_schedule(_context: dg.ScheduleEvaluationContext):
    return {}
```

### ZenML

```python
from zenml.config.schedule import Schedule


schedule = Schedule(cron_expression="0 2 * * *")
my_pipeline.with_options(schedule=schedule)()
```

**Key differences**:
- ZenML scheduling support is orchestrator-dependent.
- The current ZenML scheduling docs list support for Airflow, AzureML, Databricks, HyperAI, Kubeflow, Kubernetes, SageMaker, and Vertex, with different schedule-type support per orchestrator.
- Exact schedule APIs and supported schedule types can change; verify current ZenML docs for the target stack before promising parity.

---

## Sensor -> external trigger pattern

### Dagster

```python
import dagster as dg


@dg.sensor(job=my_job)
def file_sensor():
    if new_file_exists():
        yield dg.RunRequest(run_key="new-file")
```

### ZenML

```python
from zenml import pipeline


@pipeline
def my_pipeline(file_path: str) -> None:
    ...


def handle_file_event(file_path: str) -> None:
    # Migration note: event handling lives outside the pipeline.
    my_pipeline(file_path=file_path)
```

**Key differences**:
- The Dagster sensor becomes external eventing or trigger logic.
- Do not translate sensors into endless waiting steps unless there is no better option.

---

## Asset check -> validation step

### Dagster

```python
import dagster as dg
import pandas as pd


@dg.asset
def customers() -> pd.DataFrame:
    ...


@dg.asset_check(asset=customers)
def customers_not_empty(customers: pd.DataFrame) -> dg.AssetCheckResult:
    return dg.AssetCheckResult(passed=not customers.empty)
```

### ZenML

```python
import pandas as pd
from zenml import step


@step
def customers_not_empty(customers: pd.DataFrame) -> None:
    if customers.empty:
        raise ValueError("customers dataset is empty")
```

**Key differences**:
- The validation logic is preserved.
- The independently addressable asset-check execution model is not preserved.

---

## multi_asset -> multi-output step

### Dagster

```python
import dagster as dg


@dg.multi_asset(
    outs={
        "features": dg.AssetOut(),
        "labels": dg.AssetOut(),
    }
)
def build_training_data():
    return [1, 2, 3], [0, 1, 0]
```

### ZenML

```python
from typing import Annotated

from zenml import step


@step
def build_training_data() -> tuple[
    Annotated[list[int], "features"],
    Annotated[list[int], "labels"],
]:
    return [1, 2, 3], [0, 1, 0]
```

**Key differences**:
- This is a good fit only when both outputs are always produced together.
- If the Dagster code relies on subsetting one output without the other, flag it.

---

## graph_asset -> helper steps inside a pipeline

### Dagster

```python
import dagster as dg


@dg.op
def extract() -> list[int]:
    return [1, 2, 3]


@dg.op
def double(values: list[int]) -> list[int]:
    return [v * 2 for v in values]


@dg.graph_asset
def doubled_values() -> list[int]:
    return double(extract())
```

### ZenML

```python
from zenml import pipeline, step


@step
def extract() -> list[int]:
    return [1, 2, 3]


@step
def double(values: list[int]) -> list[int]:
    return [v * 2 for v in values]


@pipeline
def doubled_values_pipeline() -> None:
    double(extract())
```

**Key differences**:
- Dagster's graph asset exposes one asset while hiding internal ops.
- ZenML makes the steps explicit; the former asset identity becomes a pipeline/output naming choice.

---

## DynamicOutput fan-out -> dynamic pipeline or redesign

### Dagster

```python
import dagster as dg


@dg.op(out=dg.DynamicOut())
def list_regions():
    for region in ["eu", "us"]:
        yield dg.DynamicOutput(region, mapping_key=region)


@dg.op
def process_region(region: str) -> str:
    return region.upper()


@dg.job
def regional_job():
    list_regions().map(process_region)
```

### ZenML

```python
from zenml import pipeline, step


@step
def list_regions() -> list[str]:
    return ["eu", "us"]


@step
def process_region(region: str) -> str:
    return region.upper()


@pipeline(dynamic=True)
def regional_pipeline() -> None:
    regions = list_regions()
    process_region.map(regions)
```

**Key differences**:
- This is only appropriate if the target ZenML setup supports dynamic pipelines and the semantics are acceptable.
- Current ZenML docs describe dynamic-pipeline support for `local`, `local_docker`, `kubernetes`, `sagemaker`, `vertex`, and `azureml` orchestrators.
- The example syntax is illustrative; verify the current dynamic-pipeline API and stack support before treating this as a drop-in migration.
- If the original Dagster job relied on operational guarantees that the target stack cannot match, redesign instead of pretending it is equivalent.
