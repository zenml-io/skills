---
name: zenml-pipeline-authoring
description: >-
  Author ZenML pipelines: @step/@pipeline decorators, type hints, multi-output
  steps, dynamic vs static pipelines, artifact data flow, ExternalArtifact,
  YAML configuration, DockerSettings for remote execution, custom materializers,
  metadata logging, secrets management, and custom visualizations. Use this skill
  whenever asked to write a ZenML pipeline, create ZenML steps, make a pipeline
  work on Kubernetes/Vertex/SageMaker, add Docker settings, write a materializer,
  create a custom visualization, handle "works locally but fails on cloud" issues,
  or configure pipeline YAML files. Even if the user doesn't explicitly mention
  "pipeline authoring", use this skill when they ask to build an ML workflow,
  data pipeline, or training pipeline with ZenML.
---

# Author ZenML Pipelines

This skill guides pipeline authoring: steps, artifacts, configuration, Docker settings, materializers, metadata, secrets, and visualizations.

## Start Here: Interview the User

**Do not rush to code.** Before writing a single line, thoroughly understand what the user wants to build. The interview is the most important step — a well-scoped pipeline that does 3 things well beats a sprawling one that does 10 things poorly.

**For complex or multi-pipeline projects**: If the user describes something ambitious (e.g., "build me an end-to-end ML platform with data ingestion, feature engineering, training, evaluation, deployment, monitoring, and retraining"), or if they mention multiple pipelines, invoke the `zenml-scoping` skill first. It runs a deeper architectural interview that decomposes the system into pipeline units, identifies what doesn't belong in a pipeline at all, and produces a `pipeline_architecture.md` spec. Once that's done, come back here to build each pipeline one at a time.

**For single, focused pipelines**: If the user's request is clearly one pipeline (e.g., "build a training pipeline for my CSV data"), proceed with the questions below. If the answers are obvious from context, infer them and proceed. Only ask when genuinely ambiguous.

**Q1: Static or dynamic pipeline?**
Most pipelines are static (fixed DAG). Use dynamic (`@pipeline(dynamic=True)`) only when the number of steps or their wiring depends on runtime values (e.g., "process N documents where N comes from a query"). See [Dynamic Pipelines](#dynamic-pipelines) and [references/dynamic-pipelines.md](references/dynamic-pipelines.md).

**Q2: Local or remote orchestrator?**
If remote (Kubernetes, Vertex AI, SageMaker, AzureML), the [Artifact Golden Rule](#the-artifact-golden-rule) is critical, and you will need [Docker Settings](#docker-settings). If local-only for now, you can defer those concerns. Ask whether the user already has a stack set up — if not, point them to the ZenML docs for stack setup (this skill does not cover stack creation).

**Q3: Any custom Python types?**
If steps produce or consume types beyond builtins, pandas, numpy, or Pydantic models, you likely need a [custom materializer](#custom-types-and-materializers). Note: Pydantic `BaseModel` subclasses have a built-in materializer — often the simplest alternative to writing a custom materializer.

**Q4: Where should the project live?**
Ask the user where to create the project — a new subfolder, or the current directory. If the current directory is not empty, suggest a new subfolder.

**Q5: What are the data sources?**
Understand where data comes from: local CSV/Parquet files, a database (Snowflake, PostgreSQL), an API, cloud storage? This determines the first step's implementation and whether secrets are needed. If credentials are involved, always use [ZenML Secrets](#secrets-management) — never pass passwords as CLI arguments or in config files.

**Q6: Does the user want a small-data development mode?**
Many users want to iterate quickly with a fraction of the dataset. Plan for a `--sample-size` or `--small` CLI flag in `run.py`.

---

## Core Anatomy

### Defining steps

A step is a Python function decorated with `@step`. Type hints on inputs and outputs are required — they control serialization, caching, and dashboard display.

```python
from zenml import step

@step
def train_model(X_train: pd.DataFrame, lr: float = 0.01) -> sklearn.base.BaseEstimator:
    """lr is a parameter (literal value); X_train is an artifact (from upstream step)."""
    model = LogisticRegression(C=1/lr).fit(X_train.drop("target", axis=1), X_train["target"])
    return model
```

**Parameters vs artifacts**: If a step input comes from another step's output, it is an artifact. If it is a literal value passed directly (JSON-serializable), it is a parameter. ZenML handles them differently.

### Named and multi-output steps

Use `Annotated` to give outputs stable names. Use `Tuple` for multiple outputs:

```python
from typing import Annotated, Tuple
from zenml import step
import pandas as pd

@step
def split_data(df: pd.DataFrame, ratio: float = 0.8) -> Tuple[
    Annotated[pd.DataFrame, "train"],
    Annotated[pd.DataFrame, "test"],
]:
    idx = int(len(df) * ratio)
    return df.iloc[:idx], df.iloc[idx:]
```

### Wiring a pipeline

```python
from zenml import pipeline

@pipeline
def training_pipeline(dataset_path: str = "data.csv", lr: float = 0.01) -> None:
    df = load_data(path=dataset_path)
    train, test = split_data(df=df)
    model = train_model(X_train=train, lr=lr)
    evaluate(model=model, X_test=test)

if __name__ == "__main__":
    training_pipeline()
```

Pipeline parameters (like `dataset_path`) can be overridden at runtime or via YAML config.

### Step invocation IDs

When you call a step multiple times in one pipeline, ZenML auto-suffixes the name (`scale`, `scale_2`). Override with `my_step(id="custom_id")`.

### Project structure

Every pipeline project MUST follow this layout. This is non-negotiable — it produces clean, maintainable projects:

```
my_pipeline_project/
├── steps/                    # One file per step
│   ├── load_data.py
│   ├── preprocess.py
│   ├── train_model.py
│   └── evaluate.py
├── pipelines/
│   └── training.py           # Pipeline definition(s)
├── materializers/            # Custom materializers (if any)
│   └── my_data_materializer.py
├── visualizations/           # HTML/CSS templates for dashboard visualizations
│   └── metrics_report.html
├── configs/                  # One YAML config per environment
│   ├── dev.yaml
│   ├── staging.yaml
│   └── prod.yaml
├── run.py                    # CLI entry point (argparse, not click)
├── README.md                 # How to run, what stacks to use, etc.
└── pyproject.toml            # Dependencies — always pyproject.toml, not requirements.txt
```

Key rules:
- **One step per file** in a `steps/` directory — not all steps in one `steps.py`.
- **Separate pipeline definition from execution** — pipeline in `pipelines/`, execution in `run.py`.
- **Always create a `README.md`** (not `summary.md`) explaining how to run the pipeline, what stacks it supports, and any setup needed. Link to the relevant ZenML docs pages (e.g., dynamic pipelines docs) rather than embedding lengthy explanations. Do NOT include stack registration or setup instructions — just say "assumes you have a ZenML stack configured" and link to https://docs.zenml.io for stack setup.
- **Always use `pyproject.toml`** for dependency declarations. Do NOT create `requirements.txt` alongside it — use one or the other, and `pyproject.toml` is the right choice.
- **`run.py` uses `argparse`** (not `click`) — click can conflict with ZenML's own click dependency.
- **Run `zenml init`** at the project root to set the source root explicitly — this prevents import failures when code runs inside containers.

### pyproject.toml template

Always use this as the starting point for `pyproject.toml`:

```toml
[project]
name = "my-pipeline-project"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "zenml>=0.93",
    "pandas>=2.0",
    # Add pipeline-specific dependencies here
]

[project.optional-dependencies]
dev = [
    "pytest",
    "ruff",
    "mypy",
]
```

Key version constraints:
- **Python >= 3.12** — ZenML's modern features and type annotations benefit from 3.12+.
- **ZenML >= 0.93** — this is the minimum for current features. For dynamic pipelines, require >= 0.91 at absolute minimum (but 0.93 is safer).
- **Don't pin dev tool versions** (pytest, ruff, mypy) — just list them without version constraints so users get the latest.
- **Prefer `uv`** in README instructions: `uv pip install -e ".[dev]"` for faster and more reliable resolution. If `uv` is unavailable in the user's environment, use `pip install -e ".[dev]"`.

### README template notes

The README should include:
- How to install: Prefer `uv pip install -e ".[dev]"` and `zenml integration install <name> --uv` when supported. If `uv` is unavailable, use pip equivalents. Omit `-y` so users can review prompts.
- How to run: `python run.py --config configs/dev.yaml`
- What stacks it supports (just name them, don't explain how to register them)
- **Link to specific orchestrator docs** — not just generic https://docs.zenml.io. For example, if targeting Vertex AI, link to the Vertex AI orchestrator page and the GCP service connector page. Encourage users to use **service connectors** for authentication rather than manual credential management.
- A simple **ASCII DAG visualization** of the pipeline flow is a nice touch:
  ```
  load_data --> preprocess --> train_model --> evaluate
  ```

### `run.py` CLI template

Every `run.py` should offer these flags:

```python
import argparse
from pipelines.training import training_pipeline

def main():
    parser = argparse.ArgumentParser(description="Run the training pipeline")
    parser.add_argument("--config", default="configs/dev.yaml", help="Path to YAML config")
    parser.add_argument("--no-cache", action="store_true", help="Disable caching")
    parser.add_argument("--sample-size", type=int, default=None,
                        help="Use only N rows (for quick local iteration)")
    args = parser.parse_args()

    pipeline_instance = training_pipeline.with_options(
        config_path=args.config,
        enable_cache=not args.no_cache,
    )
    pipeline_instance(sample_size=args.sample_size)

if __name__ == "__main__":
    main()
```

The `sample_size` parameter is passed as a pipeline parameter so the data-loading step can slice the dataset.

---

## The Artifact Golden Rule

> **Data must enter and move through the pipeline as artifacts, not as local file paths.**

This is the single most important concept for cloud portability. When running on a remote orchestrator, each step runs in a separate container on a separate machine. There is no shared filesystem between steps.

### What goes wrong

```python
# ANTI-PATTERN: works locally, fails on cloud
@step
def preprocess(input_path: str) -> str:
    df = pd.read_csv(input_path)           # Reads from local disk
    output_path = "/tmp/processed.csv"
    df.to_csv(output_path)
    return output_path                      # Next step can't access /tmp on a different pod

@step
def train(data_path: str) -> None:
    df = pd.read_csv(data_path)             # FileNotFoundError on cloud!
```

### The correct pattern

```python
# CORRECT: data flows as artifacts
@step
def preprocess(input_path: str) -> pd.DataFrame:
    return pd.read_csv(input_path)          # ZenML serializes the DataFrame to the artifact store

@step
def train(data: pd.DataFrame) -> None:
    ...                                     # ZenML loads it from the artifact store — works everywhere
```

The first step in a pipeline is typically the one that bridges external data into the artifact world. All downstream steps receive artifacts, never file paths.

---

## Dynamic Pipelines

Use dynamic pipelines when the DAG shape depends on runtime values. They are experimental and have restricted orchestrator support (Local, LocalDocker, Kubernetes, Vertex, SageMaker, AzureML). Always link to the dynamic pipelines documentation (https://docs.zenml.io/how-to/steps-pipelines/dynamic-pipelines) in the README since these APIs can be tricky to get right.

### Minimal example

```python
from zenml import pipeline, step

@step
def get_count() -> int:
    return 3

@step
def process(index: int) -> None:
    print(f"Processing {index}")

@pipeline(dynamic=True)
def my_dynamic_pipeline() -> None:
    count = get_count()
    count_data = count.load()       # .load() gets actual Python value
    for idx in range(count_data):
        process(index=idx)
```

### The critical distinction: `.load()` vs `.chunk()`

| Method | Returns | Use for |
|--------|---------|---------|
| `.load()` | Actual Python data | Decisions, control flow, iteration |
| `.chunk(index=i)` | A DAG edge reference | Wiring to downstream steps |

You typically need both: `.load()` to iterate/decide, `.chunk()` to wire the DAG:

```python
items = produce_list()
for i, val in enumerate(items.load()):   # load to iterate
    if val > threshold:
        chunk = items.chunk(index=i)      # chunk to wire
        process(chunk)
```

### Fan-out with `.map()` and parallel execution with `.submit()`

For map/reduce patterns, `.map()` fans out over a collection. For explicit parallelism, `.submit()` returns a future.

See [references/dynamic-pipelines.md](references/dynamic-pipelines.md) for the complete API: `.map()`, `.product()`, `.submit()`, `unmapped()`, `.unpack()`, runtime modes, orchestrator support table, and limitations.

---

## Injecting External Data

When data originates outside the pipeline (a local file, a database, an API), you need to bridge it into the artifact system.

### Pattern A: `ExternalArtifact(value=...)`

Upload data inline when defining the pipeline. Simple but disables caching for the consuming step:

```python
from zenml import ExternalArtifact, pipeline, step
import pandas as pd

@step
def train(data: pd.DataFrame) -> None:
    ...

@pipeline
def my_pipeline() -> None:
    df = pd.read_csv("local_data.csv")
    train(data=ExternalArtifact(value=df))
```

### Pattern B: Pre-upload + UUID reference (for remote orchestrators)

For dynamic pipelines on remote orchestrators, the pipeline function runs inside the orchestrator pod — it cannot read your local filesystem. Pre-upload the data, then reference it by UUID:

```python
# run.py (client-side, runs on your machine)
from zenml.artifacts.utils import save_artifact
import pandas as pd

df = pd.read_csv("local_data.csv")
art = save_artifact(data=df, name="my_dataset")
print(art.id)  # Pass this UUID to the pipeline

# pipeline.py (runs inside the orchestrator pod)
from zenml.client import Client
from uuid import UUID

@pipeline
def my_pipeline(dataset_id: str) -> None:
    artifact = Client().get_artifact_version(UUID(dataset_id))
    train(data=artifact)
```

**Important**: `ExternalArtifact(id=...)` is rejected by the current validator. Use `Client().get_artifact_version()` instead.

See [references/external-data.md](references/external-data.md) for additional patterns including `register_artifact()`.

---

## YAML Configuration

Separate environment-specific settings from pipeline code using YAML config files.

### Minimal example

```yaml
# configs/dev.yaml
enable_cache: false
parameters:
  dataset_path: "data/small.csv"
  lr: 0.05
steps:
  train_model:
    settings:
      resources:
        cpu_count: 2
```

```python
training_pipeline.with_options(config_path="configs/dev.yaml")()
```

**Configuration precedence** (highest to lowest): Runtime Python code > Step-level YAML > Pipeline-level YAML > Defaults.

**Always use separate config files per environment** (`configs/dev.yaml`, `configs/staging.yaml`, `configs/prod.yaml`) — never a single `config.yaml`. Generate a template with `zenml pipeline build-configuration my_pipeline > config_template.yaml`.

Prefer `with_options()` (returns a copy) over `configure()` (mutates in place).

See [references/yaml-config.md](references/yaml-config.md) for the complete YAML schema and multi-env pattern.

---

## Docker Settings

When running on remote orchestrators, ZenML builds Docker images for each step. Use `DockerSettings` to control what goes into those images.

### Common patterns

```python
from zenml.config import DockerSettings

# Install pip packages
docker = DockerSettings(
    requirements=["scikit-learn>=1.0", "pandas>=2.0"],
    apt_packages=["libgomp1"],
    environment={"PYTHONUNBUFFERED": "1"},
)

@pipeline(settings={"docker": docker})
def my_pipeline() -> None:
    ...
```

### Step-level overrides

Apply different settings per step with `@step(settings={"docker": DockerSettings(parent_image="pytorch/pytorch:2.2.0-cuda11.8-cudnn8-runtime", requirements=["transformers"])})`.

Key fields: `requirements` (pip packages), `required_integrations` (ZenML integrations), `apt_packages` (system packages), `environment` (build-time env vars), `runtime_environment` (runtime env vars), `parent_image` (custom base image), `python_package_installer` (`"uv"` default or `"pip"`), `prevent_build_reuse` (force fresh builds for debugging).

See [references/docker-settings.md](references/docker-settings.md) for the full field catalog and YAML equivalents.

---

## Metadata Logging

Log metadata from within steps to track metrics, parameters, and results in the ZenML dashboard. This is essential for production pipelines — every training step, evaluation step, and data quality step should log relevant metadata.

### Basic usage (inside a step)

```python
from zenml import step
from zenml.utils.metadata_utils import log_metadata

@step
def train_model(X_train: pd.DataFrame, y_train: pd.Series) -> sklearn.base.BaseEstimator:
    model = LogisticRegression().fit(X_train, y_train)
    log_metadata({"accuracy": model.score(X_train, y_train), "n_samples": len(X_train)})
    return model
```

When called with only `metadata` inside a step, it automatically attaches to the current step.

### Nested metadata (creates separate cards in the dashboard)

Use nested dicts — each top-level key becomes its own card: `log_metadata({"model_metrics": {"accuracy": 0.95, "f1": 0.90}, "data_stats": {"n_samples": 5000}})`.

Log metadata in **every** data loading step (row count, source), preprocessing step (rows removed), training step (all metrics, hyperparams), and evaluation step (test metrics, thresholds).

---

## Secrets Management

When a pipeline needs credentials (database passwords, API keys, cloud tokens), **never pass them as CLI arguments, config file values, or environment variables in code**. Use the ZenML secret store:

```bash
# One-time setup (CLI)
zenml secret create db_credentials --host=db.example.com --username=admin --password=secret123
```

```python
from zenml import step
from zenml.client import Client

@step
def load_from_database(query: str) -> pd.DataFrame:
    secret = Client().get_secret("db_credentials")
    host = secret.secret_values["host"]
    username = secret.secret_values["username"]
    password = secret.secret_values["password"]
    # Use credentials to connect...
```

This works on any orchestrator — the secret store is centralized in the ZenML server.

---

## Custom Types and Materializers

### When do you need a custom materializer?

| Situation | Materializer needed? |
|-----------|---------------------|
| Step returns `int`, `str`, `float`, `bool`, `dict`, `list` | No — built-in |
| Step returns `pd.DataFrame`, `np.ndarray` | No — built-in |
| Step returns a Pydantic `BaseModel` | No — built-in. **Prefer this path for custom types** |
| Step returns a `pathlib.Path` (file or directory) | No — built-in `PathMaterializer` handles it. Archives directories as `.tar.gz`, copies single files. Great for model checkpoints, data folders, or any file-based artifact |
| Step returns a custom dataclass | **Convert to Pydantic `BaseModel`** — it has a built-in materializer, validation, and JSON serialization. Only write a custom materializer if you specifically need a non-JSON format |
| Step returns a complex domain object | **Yes** |
| You want a stable format (JSON/Parquet) instead of cloudpickle | **Yes** |

### Zero-effort visualizations

Return special string types for instant dashboard visualizations — no materializer needed:

```python
from zenml.types import HTMLString, MarkdownString, CSVString

@step
def report(metrics: dict) -> HTMLString:
    # For polished HTML, load from a separate template file rather than inline strings
    html_path = Path(__file__).parent.parent / "visualizations" / "metrics_report.html"
    template = html_path.read_text()
    return HTMLString(template.format(**metrics))
```

**Visualization quality**: Keep HTML templates in a `visualizations/` directory as separate `.html` files (optionally with embedded CSS). This makes them easy to edit and preview in a browser. Avoid writing raw HTML strings inside Python — it produces ugly, hard-to-maintain visualizations. Include proper CSS styling for a polished dashboard appearance.

### Minimal custom materializer

```python
import json, os
from typing import Any, Type
from zenml.enums import ArtifactType
from zenml.materializers.base_materializer import BaseMaterializer

class MyDataMaterializer(BaseMaterializer):
    ASSOCIATED_TYPES = (MyData,)
    ASSOCIATED_ARTIFACT_TYPE = ArtifactType.DATA

    def save(self, data: MyData) -> None:
        path = os.path.join(self.uri, "data.json")
        with self.artifact_store.open(path, "w") as f:   # Always use self.artifact_store.open()
            json.dump(data.to_dict(), f)

    def load(self, data_type: Type[Any]) -> MyData:
        path = os.path.join(self.uri, "data.json")
        with self.artifact_store.open(path, "r") as f:
            return MyData(**json.load(f))
```

Use `self.artifact_store.open()` (not plain `open()`) so it works with S3, GCS, and Azure Blob — not just local filesystems.

Assign it: `@step(output_materializers=MyDataMaterializer)`.

See [references/materializers.md](references/materializers.md) for visualizations, metadata extraction, and registration options.

---

## Logging

ZenML automatically captures `print()` and standard `logging` module output from within steps — no special setup needed. Logs are stored in the artifact store and visible in the dashboard.

```python
import logging
from zenml import step

logger = logging.getLogger(__name__)

@step
def train_model(data: pd.DataFrame) -> Model:
    logger.info(f"Training on {len(data)} samples")  # Captured by ZenML
    print("Starting training...")                      # Also captured
    ...
```

Key points:
- Only logs from the **main thread** are captured. If you spawn threads, use `contextvars.copy_context()` to propagate ZenML's logging context.
- For remote runs, set logging verbosity via DockerSettings: `DockerSettings(environment={"ZENML_LOGGING_VERBOSITY": "DEBUG"})`.
- Disable step log storage with `@step(enable_step_logs=False)` or `ZENML_DISABLE_STEP_LOGS_STORAGE=true`.
- To view logs in the dashboard with a remote artifact store, you need a configured service connector.

---

## Retry Configuration

For flaky external services or transient cloud errors, use `StepRetryConfig(max_retries=3, delay=10, backoff=2)` on `@step(retry=...)` or `@pipeline(retry=...)`. YAML equivalent uses `retry:` at step or top level. See ZenML docs for hooks (`on_failure`, `on_success`).

---

## Post-Creation Enhancements

After the pipeline is scaffolded and working, offer these enhancements. Each is optional — ask the user which they want before adding them.

### Tags (add by default)

Always add tags to pipelines and key artifacts — they make filtering and organization much easier in the dashboard. Use `@pipeline(tags=["training", "v1"])` and `ArtifactConfig(tags=["dataset"])` on step outputs. Cascade tags (`Tag(name="experiment-42", cascade=True)`) automatically propagate to all artifact versions created during the run.

### Model Control Plane

Track the pipeline's artifacts under a named model for versioning, promotion, and cross-pipeline artifact sharing. Use `@pipeline(model=Model(name="my_model", tags=["classification"]))`. This enables model promotion (`staging` → `production`) and artifact exchange between training and inference pipelines. See [references/post-creation.md](references/post-creation.md#model-control-plane) for full patterns.

### Scheduling

Run the pipeline on a recurring schedule. Use `Schedule(cron_expression="0 2 * * *")` or `Schedule(interval_second=3600)` with `pipeline.with_options(schedule=schedule)`. Not all orchestrators support scheduling — Kubernetes, Vertex AI, SageMaker, AzureML, Airflow, and Kubeflow do; Local and SkyPilot do not. See [references/post-creation.md](references/post-creation.md#scheduling) for the orchestrator support table and management commands.

### Pipeline Deployment (HTTP serving)

For real-time inference or agent workflows, pipelines can be deployed as persistent HTTP services using `pipeline.deploy(deployment_name="my_service")`. This replaces the deprecated model deployer components. For advanced deployment patterns and production configuration, see the [deployment docs](https://docs.zenml.io/how-to/deployment/deployment).

See [references/post-creation.md](references/post-creation.md) for detailed patterns for all of the above.

---

## Common Anti-Patterns

| Anti-pattern | Symptom | Fix |
|-------------|---------|-----|
| Pass local file path between steps | `FileNotFoundError` on cloud | Return data as artifact, not path |
| Write to `/tmp` and read in next step | Works locally, fails on K8s | Use artifact outputs |
| Missing type hints on step | Silent failures, no caching | Add type annotations to all inputs/outputs |
| `cloudpickle` for production artifacts | Breaks across Python versions | Write a custom `BaseMaterializer` |
| `ExternalArtifact(id=...)` | `ValueError` from validator | Use `Client().get_artifact_version()` |
| Missing `DockerSettings` deps | `ModuleNotFoundError` in container | Add to `requirements` or `required_integrations` |
| Imports inside pipeline function body | Fails in container if module not available | Import at module level |
| No `zenml init` at project root | Import errors in remote steps | Run `zenml init` to set source root |
| Passwords/secrets in CLI args or config | Security risk, visible in logs | Use `Client().get_secret()` from ZenML secret store |
| All steps in one `steps.py` | Hard to maintain, test, review | One file per step in `steps/` directory |
| No metadata logging in train/eval steps | No metrics visible in dashboard | Add `log_metadata()` calls |
| Single `config.yaml` for all environments | Config drift, manual editing | Separate `configs/dev.yaml`, `configs/prod.yaml` |
| Using `click` for `run.py` CLI | Version conflicts with ZenML | Use `argparse` instead |
| Missing or wrong dep file (`requirements.txt` instead of / alongside `pyproject.toml`) | Unclear deps, no Python version pin, no dev deps | Use `pyproject.toml` only with `zenml>=0.93`, `requires-python>=3.12`, and `[project.optional-dependencies]` for dev tools |
| Inline HTML strings in Python | Ugly visualizations, hard to edit | Use separate `.html` template files |
| Defaulting to `pip install` when `uv` is available | Slower installs and weaker dependency resolution in many environments | Prefer `uv pip install` in docs/README; fall back to pip when `uv` is unavailable |
| `zenml integration install X -y` | User can't review prompts; easier to make mistakes | Prefer `zenml integration install X --uv` (no `-y`) when available; otherwise run without `-y` using default installer |
| Stack registration instructions in README | Users have different stacks; instructions become stale | Just say "assumes a configured ZenML stack" and link to docs |
| Using `dataclass` for step outputs | No built-in materializer, requires custom code | Use Pydantic `BaseModel` — has built-in materializer |
| No minimum ZenML version in deps | Breaks on older versions missing features | Pin `zenml>=0.93` (or `>=0.91` minimum for dynamic) |

---

## Resources

### Reference files (detailed guides)

- [references/dynamic-pipelines.md](references/dynamic-pipelines.md) — Complete dynamic pipeline API
- [references/external-data.md](references/external-data.md) — Data injection patterns
- [references/docker-settings.md](references/docker-settings.md) — Full DockerSettings field catalog
- [references/materializers.md](references/materializers.md) — Materializer authoring guide
- [references/yaml-config.md](references/yaml-config.md) — Complete YAML config schema
- [references/post-creation.md](references/post-creation.md) — Tags, Model Control Plane, scheduling, deployment

### ZenML documentation

For topics not covered here (stack setup, experiment tracking, advanced deployment configuration), query the ZenML docs at https://docs.zenml.io.

When linking to docs in generated READMEs, **link to specific pages** rather than the generic homepage. Common links to include:
- Dynamic pipelines: `https://docs.zenml.io/how-to/steps-pipelines/dynamic-pipelines`
- Service connectors (for cloud auth): `https://docs.zenml.io/how-to/infrastructure-deployment/auth-management`
- Orchestrator-specific pages (e.g., Vertex AI, Kubernetes, SageMaker) — search docs for the specific orchestrator name
- Encourage **service connectors** over manual credential management for cloud stacks
