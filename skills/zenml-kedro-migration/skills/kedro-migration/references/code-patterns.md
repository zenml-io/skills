# Kedro -> ZenML Code Patterns

These examples show the **shape** of the migration, not the only valid implementation. The goal is to make the boundary shift obvious: Kedro hides IO behind the Data Catalog; ZenML makes boundary IO explicit and keeps internal edges as typed artifacts.

## Table of Contents

- [1. Simple pipeline with catalog datasets](#1-simple-pipeline-with-catalog-datasets)
- [2. Parameters from `parameters.yml` and `params:`](#2-parameters-from-parametersyml-and-params)
- [3. Multiple inputs and outputs on one node](#3-multiple-inputs-and-outputs-on-one-node)
- [4. Modular pipelines with namespacing](#4-modular-pipelines-with-namespacing)
- [5. Pipeline slicing and partial reruns](#5-pipeline-slicing-and-partial-reruns)
- [6. Versioned datasets](#6-versioned-datasets)
- [7. Credentials and secret handling](#7-credentials-and-secret-handling)
- [8. Hooks such as `before_node_run` and `on_node_error`](#8-hooks-such-as-before_node_run-and-on_node_error)
- [9. `ParallelRunner` and `ThreadRunner` semantics](#9-parallelrunner-and-threadrunner-semantics)
- [10. Experiment tracking with `kedro-mlflow`](#10-experiment-tracking-with-kedro-mlflow)
- [11. Spark datasets and transcoding](#11-spark-datasets-and-transcoding)
- [12. Free and intermediate datasets](#12-free-and-intermediate-datasets)
- [13. Modular pipeline remapping](#13-modular-pipeline-remapping)
- [14. Deployment patterns](#14-deployment-patterns)

## 1. Simple pipeline with catalog datasets

### Kedro

```yaml
# conf/base/catalog.yml
raw_customers:
  type: pandas.CSVDataset
  filepath: data/01_raw/customers.csv

customer_stats:
  type: json.JSONDataset
  filepath: data/08_reporting/customer_stats.json
```

```python
node(clean_customers, "raw_customers", "clean_customers")
node(summarize_customers, "clean_customers", "customer_stats")
```

### ZenML

```python
from typing import Annotated
import pandas as pd
from zenml import pipeline, step


@step
def load_raw_customers(path: str) -> pd.DataFrame:
    return pd.read_csv(path)


@step
def clean_customers_step(customers: pd.DataFrame) -> pd.DataFrame:
    ...


@step
def summarize_customers_step(df: pd.DataFrame) -> Annotated[dict[str, float], "customer_stats"]:
    ...


@step
def export_stats(stats: dict[str, float], output_path: str) -> str:
    ...


@pipeline
def customers_pipeline(raw_customers_path: str, customer_stats_path: str) -> None:
    raw = load_raw_customers(path=raw_customers_path)
    clean = clean_customers_step(customers=raw)
    stats = summarize_customers_step(df=clean)
    export_stats(stats=stats, output_path=customer_stats_path)
```

**Key differences**
- Reads and writes become explicit steps
- Internal datasets become artifacts, not file-path aliases

**Migration warning**
- Do not pass report file paths between transformation steps unless they are truly external boundaries.

---

## 2. Parameters from `parameters.yml` and `params:`

### Kedro

```python
node(
    filter_high_value_orders,
    inputs=["orders", "params:high_value_threshold"],
    outputs="high_value_orders",
)
```

### ZenML

```python
@step
def filter_high_value_orders_step(
    orders: pd.DataFrame,
    high_value_threshold: float,
) -> pd.DataFrame:
    ...


@pipeline
def orders_pipeline(
    orders_path: str,
    high_value_threshold: float,
) -> None:
    orders = load_orders(path=orders_path)
    filter_high_value_orders_step(
        orders=orders,
        high_value_threshold=high_value_threshold,
    )
```

```yaml
# configs/dev.yaml
parameters:
  orders_path: data/01_raw/orders.csv
  high_value_threshold: 1000.0
```

**Key differences**
- `params:` becomes a typed function parameter
- YAML config still exists, but it is not a Data Catalog

**Migration warning**
- Do not hide parameter use inside global config reads if a step signature can make the contract explicit.

---

## 3. Multiple inputs and outputs on one node

### Kedro

```python
node(
    score_customers,
    inputs={"customers": "customers", "orders": "orders", "bias": "params:bias"},
    outputs=["scored_rows", "score_summary"],
)
```

### ZenML

```python
from typing import Annotated


@step
def score_customers_step(
    customers: pd.DataFrame,
    orders: pd.DataFrame,
    bias: float,
) -> tuple[
    Annotated[pd.DataFrame, "scored_rows"],
    Annotated[dict[str, float], "score_summary"],
]:
    ...
```

**Key differences**
- The multi-output capability survives well
- Output names are artifact names, not catalog names with hidden resolution rules

**Migration warning**
- If the original project relied on output names for catalog factories or namespace tricks, this translation is incomplete without an explicit redesign.

---

## 4. Modular pipelines with namespacing

### Kedro

```python
active = pipeline(base_training, namespace="active", parameters={"params:min_rows": "params:active_min_rows"})
candidate = pipeline(base_training, namespace="candidate", parameters={"params:min_rows": "params:candidate_min_rows"})
return active + candidate
```

### ZenML

```python
def training_subgraph(features: pd.DataFrame, min_rows: int, prefix: str) -> dict[str, float]:
    train, test = split_rows_step(features=features, min_rows=min_rows, id=f"{prefix}_split")
    model = fit_rule_model_step(train=train, id=f"{prefix}_fit")
    return evaluate_rule_model_step(model=model, test=test, id=f"{prefix}_evaluate")


@pipeline
def compare_models_pipeline(
    features_path: str,
    active_min_rows: int,
    candidate_min_rows: int,
) -> None:
    features = load_features(path=features_path)
    training_subgraph(features=features, min_rows=active_min_rows, prefix="active")
    training_subgraph(features=features, min_rows=candidate_min_rows, prefix="candidate")
```

**Key differences**
- Reuse is explicit Python composition
- Invocation IDs replace hidden namespace-driven graph identity

**Migration warning**
- Never claim that namespace prefixes alone reproduce Kedro namespace semantics.

---

## 5. Pipeline slicing and partial reruns

### Kedro

```bash
kedro run --pipeline=training --from-nodes=build_features
kedro run --pipeline=training --only-missing-outputs
```

### ZenML

```python
import pandas as pd
from zenml import pipeline, step


@step
def load_saved_features(path: str) -> pd.DataFrame:
    return pd.read_parquet(path)


@pipeline
def train_from_saved_features_pipeline(features_path: str) -> None:
    features = load_saved_features(path=features_path)
    train_model_step(features=features)
```

**Key differences**
- Kedro slicing changes which part of a predefined graph runs
- ZenML usually handles this with smaller entrypoint pipelines, explicit reload steps, artifact reuse, and caching

**Migration warning**
- Do not present caching as a true equivalent of slicing. If you want cross-run reuse, design an explicit entrypoint around a stable saved artifact or an explicit external boundary.

---

## 6. Versioned datasets

### Kedro

```yaml
features:
  type: pandas.ParquetDataset
  filepath: data/05_model_input/features.parquet
  versioned: true
```

```bash
kedro run --load-versions=features:2024-06-04T09.52.53.123Z
```

### ZenML

```python
from typing import Annotated
from pathlib import Path
import pandas as pd
from zenml import ArtifactConfig


@step
def build_features_step() -> Annotated[
    pd.DataFrame,
    ArtifactConfig(name="features", version="release_2026_04_07"),
]:
    ...


@step
def export_features_version(features: pd.DataFrame, output_dir: str, version: str) -> str:
    path = Path(output_dir) / version / "features.parquet"
    path.parent.mkdir(parents=True, exist_ok=True)
    features.to_parquet(path, index=False)
    return str(path)
```

**Key differences**
- Kedro versions file-backed datasets
- ZenML versions artifacts in the lineage graph

**Migration warning**
- If downstream systems relied on concrete versioned file paths, keep exporter logic instead of relying on artifact versioning alone.

---

## 7. Credentials and secret handling

### Kedro

```yaml
# catalog.yml
orders:
  type: pandas.CSVDataset
  filepath: s3://company-bucket/orders.csv
  credentials: orders_s3
```

```yaml
# credentials.yml
orders_s3:
  key: ...
  secret: ...
```

### ZenML

```python
@step
def load_orders(path: str, aws_secret_name: str) -> pd.DataFrame:
    # Migration note: auth is now explicit and should come from ZenML secrets
    import os
    import pandas as pd

    credentials = os.environ[aws_secret_name]
    return pd.read_csv(path, storage_options={"token": credentials})
```

**Key differences**
- Auth is no longer attached to a dataset name in a global catalog
- The credential flow becomes explicit in stack config, service connectors, or secret-backed code

**Migration warning**
- Never inline `credentials.yml` values into checked-in ZenML config.

---

## 8. Hooks such as `before_node_run` and `on_node_error`

### Kedro

```python
class ProjectHooks:
    @hook_impl
    def before_node_run(self, node, catalog, inputs, is_async, session_id):
        validate_inputs(inputs)

    @hook_impl
    def on_node_error(self, error, node, catalog, inputs, is_async, session_id):
        notify(error, node.name)
```

### ZenML

```python
from zenml import step


@step(on_failure=my_failure_hook)
def transform_step(df: pd.DataFrame) -> pd.DataFrame:
    validate_inputs({"df": df})
    ...
```

**Key differences**
- ZenML can approximate success/failure hooks
- It does not offer the same before-step or dataset lifecycle hook surface

**Migration warning**
- Move preconditions into explicit validation steps or wrapper code rather than pretending a missing hook exists.

---

## 9. `ParallelRunner` and `ThreadRunner` semantics

### Kedro

```bash
kedro run --runner=ParallelRunner
```

### ZenML

```python
@pipeline
def training_pipeline() -> None:
    prepared = prepare_data_step()
    train_a_step(data=prepared)
    train_b_step(data=prepared)
```

**Key differences**
- Parallelism is now orchestrator-managed, often with isolated containers
- Shared memory and same-process assumptions usually disappear

**Migration warning**
- Re-test memory use, startup cost, and shared-state assumptions. "It runs branches in parallel" is not enough.

---

## 10. Experiment tracking with `kedro-mlflow`

### Kedro

```yaml
# conf/local/mlflow.yml
server:
  mlflow_tracking_uri: http://mlflow:5000
```

### ZenML

```python
from typing import Any
import pandas as pd


@step
def train_model_step(train_df: pd.DataFrame) -> dict[str, Any]:
    # experiment tracker is attached through the active stack
    metrics = {"row_count": len(train_df)}
    model = {"baseline_mean": float(train_df["target"].mean())}
    return {"model": model, "metrics": metrics}
```

**Key differences**
- Kedro often injects tracking through a plugin and hooks
- ZenML attaches trackers through the stack and step context

**Migration warning**
- Preserve the tracking intent, not the plugin wiring.

---

## 11. Spark datasets and transcoding

### Kedro

```yaml
features@spark:
  type: spark.SparkDataset
  filepath: s3://bucket/features

features@pandas:
  type: pandas.ParquetDataset
  filepath: s3://bucket/features
```

### ZenML

```python
@step
def load_features_spark(path: str) -> "SparkDataFrame":
    ...


@step
def spark_to_pandas_step(df: "SparkDataFrame") -> pd.DataFrame:
    ...
```

**Key differences**
- Representation switching becomes explicit conversion logic
- One canonical representation should own the contract

**Migration warning**
- Never silently collapse transcoding into a single representation without telling the user.

---

## 12. Free and intermediate datasets

### Kedro

```python
node(build_features, "raw_rows", "features")
node(train_model, "features", "model")
```

When `features` is undeclared, Kedro may materialize it as a runtime-only dataset.

### ZenML

```python
@pipeline
def training_pipeline(raw_rows_path: str) -> None:
    raw = load_raw_rows(path=raw_rows_path)
    features = build_features_step(rows=raw)
    train_model_step(features=features)
```

**Key differences**
- The natural ZenML translation persists `features` as an artifact
- This may be a feature, but it is still a behavior change

**Migration warning**
- If the original project depended on ephemerality for privacy, cost, or storage behavior, flag it explicitly.

---

## 13. Modular pipeline remapping

### Kedro

```python
pipeline(
    base_training,
    inputs={"features": "candidate_features"},
    outputs={"model": "candidate_model"},
    parameters={"params:min_rows": "params:candidate_min_rows"},
)
```

### ZenML

```python
def candidate_training_subgraph(
    candidate_features: pd.DataFrame,
    candidate_min_rows: int,
) -> dict[str, float]:
    return training_subgraph(
        features=candidate_features,
        min_rows=candidate_min_rows,
        prefix="candidate",
    )
```

**Key differences**
- Kedro remapping is declarative and dataset-name-driven
- ZenML remapping is explicit argument passing and wrapper composition

**Migration warning**
- If the Kedro project relies on heavy remapping for reuse, expect more structural rewrite.

---

## 14. Deployment patterns

### Kedro

```bash
kedro docker build
kedro airflow create
kedro kubeflow compile
```

### ZenML

```python
from zenml import pipeline
from zenml.config import DockerSettings, ResourceSettings, Schedule


docker_settings = DockerSettings(requirements=["pandas", "scikit-learn"])
resource_settings = ResourceSettings(cpu_count=2, memory="8Gi")
schedule = Schedule(cron_expression="0 2 * * *")


@pipeline(
    settings={
        "docker": docker_settings,
        "resources": resource_settings,
    }
)
def training_pipeline() -> None:
    ...


training_pipeline.with_options(schedule=schedule)()
```

**Key differences**
- Kedro often exports the project into a platform-specific artifact
- ZenML keeps one pipeline definition and changes the active stack / orchestrator

**Migration warning**
- Migrate the **deployment intent** (scheduling, containerization, resources, tracker, secrets), not the plugin command itself.

---

## General Translation Rule of Thumb

When you are unsure how to translate a Kedro pattern, ask:

1. Is this an **internal artifact edge** or an **external boundary**?
2. Is this behavior encoded in **catalog metadata**, **hook code**, or **runtime CLI habits**?
3. Does ZenML have a real equivalent, or am I about to fake one?

If the answer to the last question is "fake one", stop and flag it in the migration report.
