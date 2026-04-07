# Vertex AI Pipelines / KFP v2 → ZenML Code Patterns

Concrete translation patterns for the most common migration surfaces. These examples are intentionally opinionated: they optimize for **idiomatic ZenML**, not for preserving the KFP authoring model at any cost. Some sections are close-to-runnable skeletons; others are structural patterns that show the migration shape and the contract decisions you need to make.

## 1. Simple typed pipeline

```python
# KFP / Vertex
from kfp import dsl

@dsl.component
def load_dataset() -> dsl.Dataset:
    ...

@dsl.component
def train_model(dataset: dsl.Dataset) -> dsl.Model:
    ...

@dsl.pipeline
def training_pipeline(project: str) -> None:
    dataset = load_dataset()
    train_model(dataset=dataset.output)
```

```python
# ZenML
from typing import Annotated
import pandas as pd
from zenml import pipeline, step

@step
def load_dataset() -> Annotated[pd.DataFrame, "dataset"]:
    ...

@step
def train_model(dataset: pd.DataFrame) -> Annotated[object, "model"]:
    ...

@pipeline
def training_pipeline(project: str) -> None:
    dataset = load_dataset()
    train_model(dataset)
```

**Key differences**
- KFP artifacts are references; ZenML artifacts are materialized Python values.
- This translation is safe only when the old component treated the artifact as data, not as a meaningful URI/path.

## 2. Python function-based component translation

```python
# KFP / Vertex
from kfp import dsl

@dsl.component(
    base_image="python:3.12",
    packages_to_install=["pandas==2.2.3", "scikit-learn==1.6.1"],
)
def preprocess(input_uri: str) -> str:
    ...
```

```python
# ZenML
from zenml import step
from zenml.config import DockerSettings

@step(settings={"docker": DockerSettings(
    parent_image="python:3.12",
    requirements=["pandas==2.2.3", "scikit-learn==1.6.1"],
)})
def preprocess(input_uri: str) -> str:
    ...
```

**Translation notes**
- Move image/dependency concerns into `DockerSettings`.
- Keep portable logic in the step body; keep platform-specific infra outside it.

## 3. Container-based component translation

```python
# KFP / Vertex
from kfp.dsl import container_component, ContainerSpec

@container_component
def score_model(model_uri: str, data_uri: str):
    return ContainerSpec(
        image="gcr.io/acme/scorer:latest",
        command=["python", "/app/score.py"],
        args=["--model", model_uri, "--data", data_uri],
    )
```

```python
# ZenML
from zenml import step

@step
def score_model(model_uri: str, data_uri: str) -> str:
    # Migration note: original component was a container contract, not a normal
    # Python function. This ZenML step is a wrapper around that container logic.
    ...
```

**Migration warning**
- This is an approximation, not a direct translation.
- If the container interface is complex, non-Python, or tightly coupled to KFP placeholders, stop and redesign.

## 4. `dsl.If` / runtime branching

```python
# KFP / Vertex
from kfp import dsl

@dsl.pipeline
def pipeline():
    score = evaluate_model()
    with dsl.If(score.output > 0.9):
        deploy_model()
```

```python
# ZenML
from zenml import pipeline

@pipeline(dynamic=True)
def pipeline() -> None:
    score = evaluate_model()
    if score.load() > 0.9:
        deploy_model()
```

**Migration warning**
- Do not convert a runtime KFP condition into a plain `if` inside a static ZenML pipeline.
- `.load()` changes where the decision is made and must be treated as a semantic change worth documenting.

## 5. `dsl.ParallelFor` + `dsl.Collected`

```python
# KFP / Vertex
from kfp import dsl

@dsl.pipeline
def batch_pipeline():
    items = list_items()
    with dsl.ParallelFor(items.output) as item:
        result = process_item(item=item)
    aggregate(results=dsl.Collected(result.output))
```

```python
# ZenML
from zenml import pipeline

@pipeline(dynamic=True)
def batch_pipeline() -> None:
    items = list_items()
    results = process_item.map(item=items)
    aggregate(results)
```

**Key differences**
- `.map()` is the closest match, but not identical to KFP fan-out semantics.
- Ordering, retry behavior, and observability must be revalidated after migration.

## 6. Pipeline parameters / `PipelineJob` submission

```python
# KFP / Vertex
from google.cloud import aiplatform

job = aiplatform.PipelineJob(
    display_name="training",
    template_path="compiled.yaml",
    parameter_values={"project": "acme-prod"},
)
job.run()
```

```python
# ZenML
from zenml import pipeline

@pipeline
def training_pipeline(project: str) -> None:
    ...

training_pipeline(project="acme-prod")
```

**Translation notes**
- In a full migration, the compile boundary disappears.
- If the team must keep `template_path=...`, that is a partial migration: wrap the existing submission in a ZenML step and document the loss of step-level lineage inside the KFP sub-pipeline.

## 7. Caching configuration

```python
# KFP / Vertex
task = train_model()
task.set_caching_options(False)
```

```python
# ZenML
from zenml import step

@step(enable_cache=False)
def train_model():
    ...
```

**Key differences**
- Both systems support caching, but cache identity differs.
- Never promise that cache hits will line up exactly after migration.

## 8. Resource and GPU specification

```python
# KFP / Vertex
task = train_model()
task.set_cpu_limit("8")
task.set_memory_limit("32G")
task.set_accelerator_type("NVIDIA_TESLA_T4")
task.set_accelerator_limit(1)
```

```python
# ZenML
from zenml import step
from zenml.config import ResourceSettings

@step(settings={"resources": ResourceSettings(
    cpu_count=8,
    memory="32GB",
    gpu_count=1,
)})
def train_model():
    ...
```

**Translation notes**
- `ResourceSettings(...)` captures portable intent first.
- Add Vertex-specific orchestrator settings only for details that genuinely need them, such as machine type, service account, network, or persistent resources.

## 9. GCPC rewrite pattern: BigQuery

```python
# KFP / Vertex (structural pattern)
from google_cloud_pipeline_components.v1.bigquery import BigqueryQueryJobOp

query = BigqueryQueryJobOp(
    query="SELECT * FROM my_table",
    project="acme-prod",
)
```

```python
# ZenML
from zenml import step

@step
def run_bigquery_query(query: str, project: str) -> str:
    # Migration note: this is a rewrite of a GCPC node into ordinary SDK code.
    from google.cloud import bigquery

    client = bigquery.Client(project=project)
    job = client.query(query)
    job.result()
    return str(job.job_id)
```

**Migration warning**
- Rewrite, not translation.
- Verify auth, permissions, and output-artifact handling explicitly.

## 10. GCPC rewrite pattern: Vertex training / AutoML

```python
# KFP / Vertex (structural pattern)
from google_cloud_pipeline_components.v1.automl.training_job import (
    AutoMLTabularTrainingJobRunOp,
)

train = AutoMLTabularTrainingJobRunOp(...)
```

```python
# ZenML
from zenml import step

@step
def launch_vertex_training(project: str, region: str, dataset_uri: str) -> str:
    # Migration note: original GCPC operator has been replaced with an explicit
    # Vertex AI SDK call.
    from google.cloud import aiplatform

    aiplatform.init(project=project, location=region)
    ...
    return "vertex-training-job-id"
```

**Migration warning**
- Keep the cloud API call explicit so the user can see exactly what still depends on Vertex.
- Do not claim there is a native ZenML AutoML operator.

## 11. Endpoint deployment / model registry

```python
# ZenML
from zenml import step

@step
def upload_and_deploy_model(project: str, region: str, model_artifact_uri: str) -> str:
    from google.cloud import aiplatform

    aiplatform.init(project=project, location=region)
    # Upload model / create endpoint / deploy explicitly here.
    ...
    return "endpoint-id"
```

**Translation notes**
- ZenML model artifacts and Vertex Model Registry resources are different objects.
- Keep the handoff explicit.

## 12. Scheduling

```python
# KFP / Vertex
job.create_schedule(
    cron="TZ=Europe/Amsterdam 0 3 * * *",
    start_time="2026-04-08T00:00:00Z",
    end_time="2026-05-08T00:00:00Z",
)
```

```python
# ZenML
from datetime import datetime, timezone
from zenml.config.schedule import Schedule

training_pipeline.with_options(
    schedule=Schedule(
        cron_expression="TZ=Europe/Amsterdam 0 3 * * *",
        start_time=datetime(2026, 4, 8, tzinfo=timezone.utc),
        end_time=datetime(2026, 5, 8, tzinfo=timezone.utc),
    )
)()
```

**Migration warning**
- ZenML exposes the common subset only.
- Schedule update/delete lifecycle still needs manual validation on Vertex.

## 13. Importer / external artifact

```python
# KFP / Vertex
from kfp import dsl

existing_dataset = dsl.importer(
    artifact_uri="gs://acme-data/raw/train.parquet",
    artifact_class=dsl.Dataset,
)
```

```python
# ZenML
from zenml import ExternalArtifact, pipeline

@pipeline
def training_pipeline() -> None:
    raw_dataset = ExternalArtifact(value="gs://acme-data/raw/train.parquet")
    train_model(raw_dataset)
```

**Key differences**
- In KFP the importer registers an artifact node in the compiled graph.
- In ZenML you inject or register the external value explicitly.

## 14. Experiment tracking / metrics

```python
# ZenML
from typing import Annotated
from zenml import step

@step
def evaluate_model() -> Annotated[dict[str, float], "metrics"]:
    metrics = {"accuracy": 0.93, "f1": 0.91}
    return metrics
```

**Translation notes**
- Use artifacts and metadata logging for metrics.
- Add a Vertex experiment tracker if the user wants metrics and run links to surface in Vertex Experiments explicitly.
