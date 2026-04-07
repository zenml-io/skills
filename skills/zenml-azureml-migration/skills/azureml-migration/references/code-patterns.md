# AzureML SDK v2 -> ZenML Code Patterns

These are side-by-side translation patterns for the most common AzureML SDK v2 migration cases. Each pattern is marked as **direct**, **approximate**, or **redesign-only** so the migration does not overclaim safety.

## Table of Contents

- [Simple Pipeline with `@command_component`](#simple-pipeline-with-command_component)
- [YAML-defined Component with `load_component()`](#yaml-defined-component-with-load_component)
- [Inputs, Outputs, and Asset Types](#inputs-outputs-and-asset-types)
- [Custom Environments](#custom-environments)
- [Compute Target Specification](#compute-target-specification)
- [Sweep Jobs / HPO](#sweep-jobs--hpo)
- [Parallel Jobs / Batch Scoring](#parallel-jobs--batch-scoring)
- [Conditional Execution](#conditional-execution)
- [`do_while` Loops](#do_while-loops)
- [Sub-pipelines / Pipeline Components](#sub-pipelines--pipeline-components)
- [Scheduling](#scheduling)
- [Model Registration and Endpoint Deployment](#model-registration-and-endpoint-deployment)
- [MLflow Integration](#mlflow-integration)
- [Data Assets and Versioning](#data-assets-and-versioning)
- [Registered Components and AzureML Registry](#registered-components-and-azureml-registry)
- [AutoML](#automl)

## Simple Pipeline with `@command_component`

**Classification:** direct for the main authoring pattern, approximate for environment/compute semantics

### AzureML

```python
from mldesigner import command_component, Input, Output
from azure.ai.ml.dsl import pipeline

@command_component(name="prep_data")
def prep_data(input_data: Input(type="uri_folder"),
              training_data: Output(type="uri_folder")):
    ...

@command_component(name="train_model")
def train_model(input_data: Input(type="uri_folder"),
                model_output: Output(type="mlflow_model"),
                epochs: int = 10):
    ...

@pipeline(default_compute="cpu-cluster")
def aml_training_pipeline(pipeline_input_data):
    prep_node = prep_data(input_data=pipeline_input_data)
    train_node = train_model(input_data=prep_node.outputs.training_data, epochs=20)
    train_node.compute = "gpu-cluster"
```

### ZenML

```python
from zenml import pipeline, step
from zenml.config import DockerSettings
from zenml.integrations.azure.flavors import AzureMLOrchestratorSettings

train_docker = DockerSettings(parent_image="mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu20.04")
gpu_cluster = AzureMLOrchestratorSettings(
    mode="compute-cluster",
    compute_name="gpu-cluster",
    size="Standard_NC6s_v3",
    min_instances=0,
    max_instances=4,
)

@step
def prep_step(input_data_uri: str) -> str:
    ...

@step(settings={"docker": train_docker})
def train_step(training_data_uri: str, epochs: int = 10) -> str:
    ...

@pipeline(settings={"orchestrator": gpu_cluster})
def zenml_training_pipeline(input_data_uri: str, epochs: int = 20) -> str:
    prepared = prep_step(input_data_uri=input_data_uri)
    return train_step(training_data_uri=prepared, epochs=epochs)
```

**Key differences**
- AzureML separates component, environment, and compute. ZenML collapses those into `@step`, `DockerSettings`, and orchestrator settings.
- `node.compute = ...` is not a literal per-node DSL property in ZenML.

## YAML-defined Component with `load_component()`

**Classification:** approximate

### AzureML

```python
from azure.ai.ml import load_component

score_component = load_component(source="./components/score.yml")

@pipeline
def scoring_pipeline(data_input):
    score_component(input_data=data_input)
```

### ZenML

```python
from zenml import pipeline, step

@step
def score_step(input_data_uri: str) -> str:
    # Rewrite the YAML-defined interface as a normal Python step.
    ...

@pipeline
def scoring_pipeline(data_input_uri: str) -> str:
    return score_step(input_data_uri=data_input_uri)
```

**Migration warning**
- ZenML has YAML configuration, but not YAML-defined reusable pipeline components as a first-class asset type.
- Rewrite the component contract in Python rather than trying to preserve the YAML component artifact.

## Inputs, Outputs, and Asset Types

**Classification:** approximate

### AzureML

```python
from mldesigner import command_component, Input, Output

@command_component
def transform_data(
    raw_data: Input(type="mltable"),
    output_data: Output(type="uri_folder"),
    split: float = 0.2,
):
    ...
```

### ZenML

```python
from pandas import DataFrame
from zenml import step

@step
def transform_data(raw_data_uri: str, split: float = 0.2) -> tuple[DataFrame, DataFrame]:
    # Keep the Azure asset boundary explicit if MLTable identity matters.
    ...
```

**Key differences**
- `mltable` is not a first-class ZenML type.
- Decide whether to keep asset IDs/URIs explicit or load the data into Python objects.

## Custom Environments

**Classification:** approximate

### AzureML

```python
from azure.ai.ml.entities import Environment

env = Environment(
    image="mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu20.04",
    conda_file="./conda.yaml",
)
```

### ZenML

```python
from zenml.config import DockerSettings

docker_settings = DockerSettings(
    parent_image="mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu20.04",
    requirements=["pandas", "scikit-learn"],
)
```

**Migration warning**
- AzureML environment assets have registry/versioning semantics that ZenML does not replicate.
- Preserve custom Docker build details, but do not pretend environment assets survive unchanged.

## Compute Target Specification

**Classification:** direct for AzureML orchestrator compute modes, approximate for per-node heterogeneity

### AzureML

```python
@pipeline(default_compute="cpu-cluster")
def training_pipeline(data):
    train_node = train_component(input_data=data)
    train_node.compute = "gpu-cluster"
```

### ZenML

```python
from zenml.integrations.azure.flavors import AzureMLOrchestratorSettings

gpu_cluster = AzureMLOrchestratorSettings(
    mode="compute-cluster",
    compute_name="gpu-cluster",
    size="Standard_NC6s_v3",
    min_instances=0,
    max_instances=8,
)

@pipeline(settings={"orchestrator": gpu_cluster})
def training_pipeline(data_uri: str) -> str:
    return train_step(data_uri)
```

**Migration warning**
- Pipeline-wide compute is a good fit.
- Strongly heterogeneous per-node compute usually needs separate pipelines or explicit runtime configuration.

## Sweep Jobs / HPO

**Classification:** redesign-only

### AzureML

```python
sweep_job = train_component.sweep(
    sampling_algorithm="random",
    primary_metric="accuracy",
)
```

### ZenML

```python
@step
def run_hpo(search_space: dict[str, list[float]]) -> dict[str, float]:
    # Use Optuna or call the Azure sweep API explicitly from here.
    ...
```

**Migration warning**
- ZenML has no native sweep-job primitive.
- Either keep Azure sweep native, or redesign HPO explicitly.

## Parallel Jobs / Batch Scoring

**Classification:** redesign-only

### AzureML

```python
from azure.ai.ml.parallel import parallel_run_function

parallel_component = parallel_run_function(...)
```

### ZenML

```python
@pipeline(dynamic=True)
def scoring_pipeline(batch_uris: list[str]) -> None:
    # Only use this after validating semantics, concurrency, and failure behavior.
    ...
```

**Migration warning**
- Azure parallel jobs are not just "fan out over a list".
- Treat this as manual-review-only unless the user explicitly accepts a redesign.

## Conditional Execution

**Classification:** redesign-only

### AzureML

```python
# AzureML helper-based conditional flow
if_else(...)
```

### ZenML

```python
@pipeline(dynamic=True)
def conditional_pipeline(flag: bool) -> None:
    # Redesign deliberately. Do not claim 1:1 translation.
    ...
```

**Migration warning**
- `if_else` is treated as unsafe for automatic translation in this skill.

## `do_while` Loops

**Classification:** redesign-only

### AzureML

```python
do_while(...)
```

### ZenML

```python
@step
def run_until_converged(initial_value: float) -> float:
    # Keep the loop inside one step, or redesign as a dynamic pipeline.
    ...
```

**Migration warning**
- Treat `do_while` as manual-review-only.

## Sub-pipelines / Pipeline Components

**Classification:** approximate

### AzureML

```python
@pipeline
def prep_pipeline(data):
    ...

@pipeline
def full_pipeline(data):
    prep = prep_pipeline(data)
    ...
```

### ZenML

```python
from zenml import pipeline

@pipeline
def prep_pipeline(data_uri: str) -> str:
    ...

@pipeline
def full_pipeline(data_uri: str) -> str:
    prepared = prep_pipeline(data_uri)
    ...
```

**Key differences**
- Composition exists in both worlds.
- Registered pipeline component asset semantics do not.

## Scheduling

**Classification:** direct with lifecycle caveat

### AzureML

```python
from azure.ai.ml.entities import JobSchedule, CronTrigger

schedule = JobSchedule(
    trigger=CronTrigger(expression="*/5 * * * *"),
    create_job=pipeline_job,
)
```

### ZenML

```python
from datetime import datetime, timezone
from zenml.config.schedule import Schedule

cron_schedule = Schedule(cron_expression="*/5 * * * *")
daily_interval = Schedule(
    start_time=datetime(2026, 4, 8, 10, 15, tzinfo=timezone.utc),
    interval_second=24 * 60 * 60,
)
```

**Migration warning**
- ZenML can create AzureML cron/interval schedules when using the AzureML orchestrator.
- Updating or deleting those schedules later is not fully managed through ZenML.

## Model Registration and Endpoint Deployment

**Classification:** approximate for registration, redesign-only for managed endpoints

### AzureML

```python
from azure.ai.ml.entities import ManagedOnlineDeployment, ManagedOnlineEndpoint

endpoint = ManagedOnlineEndpoint(name="fraud-endpoint")
deployment = ManagedOnlineDeployment(name="blue", endpoint_name=endpoint.name, model=model)
```

### ZenML

```python
@step
def register_model_in_azure(model_uri: str) -> str:
    # Use Azure SDK here if Azure endpoint deployment must stay part of the workflow.
    ...
```

**Migration warning**
- ZenML has first-class model objects, but not native Azure managed endpoint primitives.
- Keep endpoint deployment Azure-native, or call Azure SDK explicitly from a ZenML step.

## MLflow Integration

**Classification:** direct

### AzureML

```python
import mlflow

mlflow.log_metric("accuracy", 0.93)
```

### ZenML

```python
import mlflow
from zenml import step

@step
def evaluate_model() -> float:
    mlflow.log_metric("accuracy", 0.93)
    return 0.93
```

**Key differences**
- The MLflow API usage can stay familiar.
- The surrounding orchestration, lineage, and model management surface changes.

## Data Assets and Versioning

**Classification:** approximate

### AzureML

```python
from azure.ai.ml import Input

data_input = Input(type="mltable", path="azureml:customer_data:12")
```

### ZenML

```python
@step
def load_training_data(data_asset_ref: str) -> str:
    # Keep the Azure asset reference explicit if version identity matters.
    return data_asset_ref
```

**Migration warning**
- If the asset version is part of governance or cross-workspace sharing, keep that Azure identity explicit.
- If not, consider migrating the boundary to ZenML artifacts.

## Registered Components and AzureML Registry

**Classification:** redesign-only

### AzureML

```python
component = ml_client.components.get(name="score_component", version="3")
```

### ZenML

```python
# Repackage the logic as importable Python code, or keep Azure Registry usage external.
```

**Migration warning**
- ZenML has no cross-workspace Azure Registry equivalent.
- Do not pretend registry-backed sharing semantics survive unchanged.

## AutoML

**Classification:** approximate

### AzureML

```python
# Azure AutoML job used inside a broader AzureML workflow
```

### ZenML

```python
@step
def run_azure_automl(config: dict[str, str]) -> str:
    # Keep Azure AutoML as an explicit external call, or replace it with
    # transparent training + HPO steps.
    ...
```

**Migration warning**
- AutoML can stay reachable from ZenML, but it is not a native ZenML primitive.
