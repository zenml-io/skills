# External Data Injection Reference

How to get data that originates outside a pipeline (local files, databases, APIs) into the ZenML artifact system.

## Why this matters

When running on a remote orchestrator (Kubernetes, Vertex AI, SageMaker), each step runs in a separate container. There is no shared filesystem. Data must be in the artifact store for steps to access it.

For **static pipelines**, the pipeline function runs on the client (your machine), so `ExternalArtifact(value=...)` can read local files and upload them.

For **dynamic pipelines on remote orchestrators**, the pipeline function runs inside the orchestrator pod — not on your machine. Local file reads in the pipeline function will fail. Pre-upload data before pipeline submission.

## Pattern A: `ExternalArtifact(value=...)`

Best for: static pipelines, small data, prototyping.

```python
from zenml import ExternalArtifact, pipeline, step
import pandas as pd

@step
def train(data: pd.DataFrame) -> None:
    ...

@pipeline
def my_pipeline(csv_path: str = "data.csv") -> None:
    df = pd.read_csv(csv_path)
    train(data=ExternalArtifact(value=df))
```

**How it works**: ZenML serializes the value to the artifact store before the pipeline runs, then passes a UUID reference to the step.

**Caveats**:
- Disables caching for the consuming step (every run re-uploads)
- The local file must be accessible when the pipeline function executes
- For dynamic pipelines on remote orchestrators, the pipeline function runs in a pod — local files are not available

### Custom materializer for ExternalArtifact

```python
ExternalArtifact(
    value=my_custom_obj,
    materializer=MyCustomMaterializer,
)
```

## Pattern B: Pre-upload + UUID reference

Best for: dynamic pipelines on remote orchestrators, large data, reusable datasets.

### Step 1: Upload on the client (your machine)

```python
# run.py
from zenml.artifacts.utils import save_artifact
import pandas as pd

df = pd.read_csv("local_data.csv")
art = save_artifact(data=df, name="training_dataset")
print(f"Artifact ID: {art.id}")
# Pass this UUID as a pipeline parameter
```

### Step 2: Reference in the pipeline (runs in pod)

```python
from uuid import UUID
from zenml import pipeline
from zenml.client import Client

@pipeline
def my_pipeline(dataset_id: str) -> None:
    artifact = Client().get_artifact_version(UUID(dataset_id))
    train(data=artifact)

if __name__ == "__main__":
    my_pipeline(dataset_id="<artifact-version-UUID-from-step-1>")
```

This works because `Client().get_artifact_version(...)` resolves the artifact version through the ZenML server, and ZenML's late materialization passes that existing artifact to the step.

**Portability caveat**: for remote runs, make sure the target stack can read the artifact's artifact store. If you saved the artifact with a local stack and then run a remote stack that uses S3/GCS/Azure Blob, the remote run may not be able to load it. Save/register the artifact with the artifact store the target stack will use.

## Pattern C: Register pre-existing cloud storage data

If your data already exists in S3/GCS/Azure Blob (the same artifact store your stack uses), register it without re-uploading:

```python
from zenml import register_artifact

register_artifact(
    folder_or_file_uri="s3://my-bucket/datasets/training-v2/",
    name="training_dataset_v2",
)
```

Then fetch the artifact version with the client and pass that artifact version into the step:

```python
from zenml.client import Client

artifact = Client().get_artifact_version("training_dataset_v2")
```

## Pattern D: First step as data loader

The simplest approach for data from APIs, databases, or remote storage — make the first step do the loading:

```python
@step
def load_from_api(endpoint: str) -> pd.DataFrame:
    """First step: bridges external data into the artifact system."""
    response = requests.get(endpoint)
    return pd.DataFrame(response.json())

@pipeline
def my_pipeline(api_url: str) -> None:
    data = load_from_api(endpoint=api_url)
    process(data=data)   # All downstream steps receive artifacts
```

This is the most portable pattern — works identically on local and remote orchestrators.

## Do not construct `ExternalArtifact(id=...)` in user code

The public `ExternalArtifact` class is for by-value uploads (`ExternalArtifact(value=...)`). Its runtime config can carry an ID after ZenML uploads the value, but user code should not initialize it with `id=...`.

For existing artifacts, use `Client().get_artifact_version(...)` instead. That method supports artifact-version UUIDs, artifact names, and artifact name + version:

```python
from uuid import UUID
from zenml.client import Client

client = Client()
by_id = client.get_artifact_version(UUID("3a92ae32-a764-4420-98ba-07da8f742b76"))
latest_by_name = client.get_artifact_version("iris_dataset")
by_name_and_version = client.get_artifact_version("iris_dataset", version="raw_2023")
```

What is stale:
- `ExternalArtifact(name=...)`
- `ExternalArtifact(version=...)`
- `ExternalArtifact(model=...)`

Those lookup fields are removed from `ExternalArtifact`. If you need lookup by name, version, or model, use `Client()` methods first, then pass the resolved artifact version into the step.

Concrete story: `ExternalArtifact(value=...)` means "upload this value as a new artifact." `Client().get_artifact_version(...)` means "reuse an artifact that already exists." Keep those two jobs separate.

## Saving artifacts outside pipelines

```python
from zenml.artifacts.utils import save_artifact, load_artifact

# Save
save_artifact(data=my_array, name="my_array")

# Load (latest version)
data = load_artifact("my_array")

# Load specific version
from zenml.client import Client
artifact = Client().get_artifact_version("my_array", "42")
data = artifact.load()
```
