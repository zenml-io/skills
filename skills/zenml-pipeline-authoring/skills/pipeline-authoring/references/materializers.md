# Custom Materializers Reference

A materializer defines how a Python object is serialized to the artifact store, loaded back, visualized in the dashboard, and what metadata is extracted.

## When you need a custom materializer

| Situation | Action |
|-----------|--------|
| Built-in types (`int`, `str`, `dict`, `list`, `pd.DataFrame`, `np.ndarray`) | No materializer needed |
| Pydantic `BaseModel` subclass | No materializer needed |
| Integration types (PyTorch models, HuggingFace datasets, sklearn models) | Install the integration; materializer is auto-registered |
| Custom dataclass or domain object | Write a custom materializer |
| You want a stable format (JSON, Parquet) instead of cloudpickle | Write a custom materializer |

**Never use `cloudpickle` for production artifacts** — it breaks across Python versions. Write a custom materializer with an explicit format.

## Full materializer template

```python
import json
import os
from typing import Any, Dict, Optional, Type

from zenml.enums import ArtifactType, VisualizationType
from zenml.materializers.base_materializer import BaseMaterializer
from zenml.metadata.metadata_types import MetadataType


class MyDataMaterializer(BaseMaterializer):
    """Materializer for MyData objects, stored as JSON."""

    ASSOCIATED_TYPES = (MyData,)
    ASSOCIATED_ARTIFACT_TYPE = ArtifactType.DATA  # or MODEL, SCHEMA, STATISTICS, SERVICE

    def save(self, data: MyData) -> None:
        """Serialize to the artifact store."""
        filepath = os.path.join(self.uri, "data.json")
        with self.artifact_store.open(filepath, "w") as f:
            json.dump(data.to_dict(), f)

    def load(self, data_type: Type[Any]) -> MyData:
        """Deserialize from the artifact store."""
        filepath = os.path.join(self.uri, "data.json")
        with self.artifact_store.open(filepath, "r") as f:
            return MyData(**json.load(f))

    def save_visualizations(self, data: MyData) -> Dict[str, VisualizationType]:
        """Optional: Generate dashboard visualizations."""
        html_path = os.path.join(self.uri, "report.html")
        with self.artifact_store.open(html_path, "w") as f:
            f.write(f"<h1>{data.name}</h1><p>Records: {len(data.records)}</p>")
        return {html_path: VisualizationType.HTML}

    def extract_metadata(self, data: MyData) -> Dict[str, MetadataType]:
        """Optional: Extract searchable metadata shown in the dashboard."""
        return {
            "num_records": len(data.records),
            "name": data.name,
        }
```

## Key rules

### Always use `self.artifact_store.open()`

Never use plain `open()` — it only works on local filesystems. `self.artifact_store.open()` works with S3, GCS, Azure Blob, and local.

```python
# WRONG: breaks on cloud artifact stores
with open(os.path.join(self.uri, "data.json"), "w") as f: ...

# CORRECT: works everywhere
with self.artifact_store.open(os.path.join(self.uri, "data.json"), "w") as f: ...
```

### `self.uri` is your directory

Each artifact gets a unique directory in the artifact store. Write files within `self.uri`. You can write multiple files (data + visualization + metadata).

## Visualization types

Supported `VisualizationType` values for `save_visualizations()`:

| Type | Content |
|------|---------|
| `VisualizationType.HTML` | HTML string or file |
| `VisualizationType.IMAGE` | PNG, JPEG, etc. |
| `VisualizationType.CSV` | CSV data |
| `VisualizationType.MARKDOWN` | Markdown text |

### Zero-effort alternative: special return types

If you just want a visualization without writing a materializer, return these types directly from a step:

```python
from zenml.types import HTMLString, MarkdownString, CSVString

@step
def html_report(metrics: dict) -> HTMLString:
    rows = "".join(f"<tr><td>{k}</td><td>{v:.3f}</td></tr>" for k, v in metrics.items())
    return HTMLString(f"<table><tr><th>Metric</th><th>Value</th></tr>{rows}</table>")

@step
def csv_export(data: list) -> CSVString:
    return CSVString("col_a,col_b\n" + "\n".join(f"{a},{b}" for a, b in data))
```

### Matplotlib to HTML

```python
import base64, io, matplotlib.pyplot as plt
from zenml.types import HTMLString

@step
def plot_step(values: list[float]) -> HTMLString:
    fig, ax = plt.subplots()
    ax.plot(values)
    buf = io.BytesIO()
    fig.savefig(buf, format="png")
    plt.close(fig)
    b64 = base64.b64encode(buf.getvalue()).decode()
    return HTMLString(f'<img src="data:image/png;base64,{b64}">')
```

## Registration methods

### Automatic (most common)

When Python imports the materializer class, it auto-registers for all types in `ASSOCIATED_TYPES`. Just ensure the module is imported.

### Per-step

```python
@step(output_materializers=MyDataMaterializer)
def my_step() -> MyData: ...

# Per-output (for multi-output steps)
@step(output_materializers={"train": TrainMaterializer, "test": TestMaterializer})
def split() -> Tuple[Annotated[A, "train"], Annotated[B, "test"]]: ...
```

### Global override

Replace the default materializer for a type across the entire project:

```python
from zenml.materializers.materializer_registry import materializer_registry

materializer_registry.register_and_overwrite_type(
    key=pd.DataFrame,
    type_=FastParquetMaterializer,
)
```

## UnmaterializedArtifact

Skip deserialization entirely and get the raw artifact store URI:

```python
from zenml.artifacts.unmaterialized_artifact import UnmaterializedArtifact

@step
def process_raw(artifact: UnmaterializedArtifact) -> None:
    print(artifact.uri)  # Direct path in the artifact store
    # Useful for tools that need file paths, not Python objects
```

## Temporary directory helper

For materializers that need to download files to a local temp directory before processing:

```python
def load(self, data_type: Type[Any]) -> MyData:
    with self.get_temporary_directory(
        delete_at_exit=False,
        delete_after_step_execution=True,
    ) as temp_dir:
        # Download from artifact store to temp_dir, then process
        local_path = os.path.join(temp_dir, "model.bin")
        # ... download and load ...
```

## Base class for abstract materializers

If you have a base materializer that should not be registered itself:

```python
class BaseCustomMaterializer(BaseMaterializer):
    SKIP_REGISTRATION = True
    ASSOCIATED_TYPES = ()
```

Only concrete subclasses will be registered.
