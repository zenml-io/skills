# Post-Creation Enhancements Reference

After scaffolding a working pipeline, these features add production-readiness. Each is independent — add only what the user needs.

## Tags

Tags organize pipelines, runs, artifacts, and models in the ZenML dashboard for easy filtering and discovery.

### Adding tags to pipelines and runs

```python
from zenml import pipeline, Tag

# Simple tags on the pipeline run
@pipeline(tags=["training", "v1", "team-ml"])
def training_pipeline() -> None:
    ...

# Exclusive tags — only one run per pipeline can hold this tag at a time
# When a new run gets it, it's automatically removed from the previous run
@pipeline(tags=["training", Tag(name="latest-training", exclusive=True)])
def training_pipeline() -> None:
    ...
```

### Adding tags to artifact versions

```python
from typing import Annotated
from zenml import step, ArtifactConfig, add_tags
import pandas as pd

# Via ArtifactConfig on step output
@step
def load_data() -> Annotated[pd.DataFrame, ArtifactConfig(name="raw_data", tags=["dataset", "v1"])]:
    ...

# Dynamically inside a step
@step
def train_model(data: pd.DataFrame) -> Model:
    add_tags(tags=["production-ready"], infer_artifact=True)
    ...
```

### Cascade tags

Cascade tags automatically propagate from the pipeline run to all artifact versions created during that run:

```python
from zenml import pipeline, Tag

@pipeline(tags=[
    "normal-tag",                              # Only on the run
    Tag(name="experiment-42", cascade=True),    # Also applied to all artifacts
])
def my_pipeline() -> None:
    ...
```

### Tagging models

```python
from zenml import Model

model = Model(
    name="iris_classifier",
    version="1.0.0",
    tags=["classification", "sklearn", "production"],
)

@pipeline(model=model)
def training_pipeline() -> None:
    ...
```

### Removing tags

```python
from zenml.utils.tag_utils import remove_tags

remove_tags(tags=["old-tag"], pipeline="my_pipeline")
remove_tags(tags=["deprecated"], artifact="my_artifact")
```

---

## Model Control Plane

The Model Control Plane tracks ML models as first-class entities — grouping artifacts, pipelines, and metadata under a named model with versions and promotion stages.

### Associating a pipeline with a model

```python
from zenml import pipeline, Model

@pipeline(
    model=Model(
        name="iris_classifier",
        description="Iris flower classification model",
        tags=["classification", "sklearn"],
    )
)
def training_pipeline() -> None:
    ...
```

Each pipeline run creates a new model version. You can explicitly name versions:

```python
@pipeline(model=Model(name="iris_classifier", version="1.0.5"))
def training_pipeline() -> None:
    ...
```

Or use templated naming:

```python
@pipeline(model=Model(name="iris_classifier", version="run-{run.id[:8]}"))
def training_pipeline() -> None:
    ...
```

### Logging metrics to the model

```python
from zenml import step
from zenml.utils.metadata_utils import log_metadata

@step
def evaluate_model(model, test_data) -> float:
    accuracy = model.score(test_data.X, test_data.y)
    log_metadata(
        metadata={
            "evaluation_metrics": {
                "accuracy": accuracy,
                "n_test_samples": len(test_data),
            }
        },
        infer_model=True,  # Attaches to the model in the current step context
    )
    return accuracy
```

### Model promotion (staging → production)

```python
from zenml import Model
from zenml.enums import ModelStages

# Promote a specific version to production
model = Model(name="iris_classifier", version="1.2.3")
model.set_stage(stage=ModelStages.PRODUCTION)

# Stages: STAGING, PRODUCTION, LATEST (virtual), ARCHIVED
```

### Sharing artifacts between pipelines via the model

The most powerful pattern — an inference pipeline can load artifacts from the production version of a training pipeline:

```python
from zenml import pipeline, step, get_pipeline_context, Model
from zenml.enums import ModelStages

@pipeline(model=Model(name="iris_classifier", version=ModelStages.PRODUCTION))
def inference_pipeline() -> None:
    model = get_pipeline_context().model
    trained_model = model.get_model_artifact("trained_model")
    predict(model=trained_model, data=load_data())
```

### Fetching model metadata

```python
from zenml.client import Client

model_version = Client().get_model_version("iris_classifier", "1.2.3")
metrics = model_version.run_metadata["evaluation_metrics"].value
print(f"Accuracy: {metrics['accuracy']}")
```

---

## Scheduling

Run pipelines on recurring schedules. Schedules are managed through the orchestrator — not all orchestrators support them.

### Orchestrator support

| Orchestrator | Scheduling | Types | Native Management |
|---|---|---|---|
| Kubernetes | Yes | Cron | Yes |
| Vertex AI | Yes | Cron | No |
| SageMaker | Yes | Cron, Interval, One-time | No |
| AzureML | Yes | Cron, Interval | No |
| Airflow | Yes | Cron, Interval | No |
| Kubeflow | Yes | Cron, Interval | No |
| Databricks | Yes | Cron only | No |
| Local / LocalDocker | No | — | — |
| SkyPilot | No | — | — |
| Tekton | No | — | — |

"Native Management" means `zenml pipeline schedule update/delete` directly modifies the orchestrator's schedule (e.g., Kubernetes CronJobs). For other orchestrators, you manage the schedule on the orchestrator side.

### Setting a schedule

```python
from zenml.config.schedule import Schedule
from datetime import datetime

# Cron expression (every day at 2 AM)
schedule = Schedule(cron_expression="0 2 * * *")

# Or interval-based (every 30 minutes)
schedule = Schedule(start_time=datetime.now(), interval_second=1800)

my_pipeline.with_options(schedule=schedule)()
```

### YAML configuration

```yaml
# configs/prod.yaml
schedule:
  cron_expression: "0 2 * * *"
```

### Managing schedules (CLI)

```bash
# List all schedules
zenml pipeline schedule list

# Update cron expression
zenml pipeline schedule update <SCHEDULE_NAME> --cron-expression='0 3 * * *'

# Pause/resume
zenml pipeline schedule deactivate <SCHEDULE_NAME>
zenml pipeline schedule activate <SCHEDULE_NAME>

# Delete (soft delete by default, preserves history)
zenml pipeline schedule delete <SCHEDULE_NAME>

# Hard delete (removes all references)
zenml pipeline schedule delete <SCHEDULE_NAME> --hard
```

---

## Pipeline Deployment (HTTP Serving)

Pipeline deployment creates persistent HTTP servers that wrap your pipeline for real-time, request-response interactions. This replaces the deprecated Model Deployer stack components.

Use this as a quick-start pattern. For advanced options (auth, middleware, custom routes, SPA hosting), see the [deployment docs](https://docs.zenml.io/how-to/deployment/deployment).

### Quick start

```python
@pipeline
def weather_agent(city: str = "Paris", temperature: float = 20) -> str:
    return process_weather(city=city, temperature=temperature)

# Deploy as an HTTP service
deployment = weather_agent.deploy(deployment_name="weather_service")
print(f"URL: {deployment.url}")
```

### Requirements for deployable pipelines

- All input parameters must have **default values**
- Input parameters must be **JSON-serializable** (int, float, str, bool, list, dict, Pydantic models)
- Pipeline should **return meaningful values** for HTTP responses
- A **Deployer** stack component must be in the active stack

### Invoking a deployment

Prefer the ZenML CLI for portability, then use HTTP calls when needed.

```bash
# CLI (portable, recommended)
zenml deployment invoke weather_service --city="London" --temperature=20

# HTTP with curl (if curl is available)
curl -X POST http://localhost:8000/invoke \
  -H "Content-Type: application/json" \
  -d '{"parameters": {"city": "London", "temperature": 20}}'
```

```python
# HTTP via Python (fallback when curl is unavailable)
import requests

response = requests.post(
    "http://localhost:8000/invoke",
    json={"parameters": {"city": "London", "temperature": 20}},
    timeout=30,
)
response.raise_for_status()
print(response.json())
```

### Use cases

- **Online ML inference**: Real-time predictions (fraud detection, recommendations)
- **LLM agent workflows**: Multi-step AI reasoning with RAG, tools, guardrails
- **Real-time data processing**: Streaming analytics, anomaly detection

For full deployment configuration (CORS, custom endpoints, middleware, authentication, SPA hosting, DeploymentSettings), see the [deployment docs](https://docs.zenml.io/how-to/deployment/deployment).
