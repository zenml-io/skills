# YAML Configuration Reference

ZenML pipelines can be configured via YAML files applied at runtime with `with_options(config_path="...")`. This separates environment-specific settings from pipeline code.

## Applying a config

```python
my_pipeline.with_options(config_path="configs/dev.yaml")()
```

## Generating a template

```bash
zenml pipeline build-configuration my_pipeline > config_template.yaml
```

Or from Python:

```python
my_pipeline.write_run_configuration_template(path="config_template.yaml")
```

## Complete YAML schema

```yaml
# ── Run identity ─────────────────────────────────
run_name: "training_run_{date}_{time}"    # Placeholders: {date}, {time}

# ── Feature flags (all Optional[bool]) ───────────
enable_cache: true
enable_artifact_metadata: true
enable_artifact_visualization: true
enable_step_logs: true
enable_pipeline_logs: false
enable_heartbeat: true

# ── Pipeline parameters ─────────────────────────
# These are passed to the @pipeline function
parameters:
  dataset_path: "data/full.csv"
  learning_rate: 0.01
  batch_size: 32

# ── Environment variables for all steps ──────────
environment:
  MY_VAR: "my_value"

# ── ZenML secrets injected as env vars ───────────
secrets:
  - my_db_credentials
  - api_keys

# ── Tags applied to the run ─────────────────────
tags:
  - experiment
  - v3

# ── Settings (applied pipeline-wide) ────────────
settings:
  docker:
    parent_image: "my-registry/my-image:latest"
    requirements:
      - torch==2.0.0
      - transformers>=4.0.0
    required_integrations:
      - mlflow
    apt_packages:
      - libgomp1
    environment:
      PYTHONUNBUFFERED: "1"
    runtime_environment:
      API_KEY: ${MY_API_KEY}
    python_package_installer: uv

  resources:
    cpu_count: 4
    gpu_count: 1
    memory: "16GB"      # Format: [0-9]+(KB|KiB|MB|MiB|GB|GiB|TB|TiB|PB|PiB)

  # Stack component settings: <category>.<flavor>
  orchestrator.kubeflow:
    synchronous: false
  experiment_tracker.mlflow:
    experiment_name: "my_experiment"

# ── Step-level overrides ─────────────────────────
steps:
  train_model:
    parameters:
      learning_rate: 0.001            # Overrides pipeline-level parameter
    enable_cache: false
    experiment_tracker: "mlflow_tracker"   # or false to disable
    step_operator: "vertex_gpu"           # or false to disable
    settings:
      docker:
        requirements:
          - cuda-toolkit
      resources:
        cpu_count: 8
        gpu_count: 2
        memory: "64GB"
    retry:
      max_retries: 3
      delay: 10
      backoff: 2

# ── Global retry (applies to all steps unless overridden) ──
retry:
  max_retries: 2
  delay: 5
  backoff: 2

# ── Scheduling ───────────────────────────────────
schedule:
  cron_expression: "0 0 * * *"        # Daily at midnight
  start_time: "2024-01-01T00:00:00Z"
  end_time: "2025-01-01T00:00:00Z"
  catchup: false
  # Alternative: interval_second: 3600  # Every hour

# ── Model Control Plane ──────────────────────────
model:
  name: "my_model"
  version: "production"               # string or int
  description: "Production training model"
  tags:
    - prod
    - v2

# ── Build override ───────────────────────────────
build: "<build_uuid>"                  # Reuse a pre-built Docker image

# ── Execution mode ───────────────────────────────
execution_mode: "CONTINUE_ON_FAILURE"  # or STOP_ON_FAILURE or FAIL_FAST

# ── Placeholder substitutions ────────────────────
substitutions:
  experiment_name: "resnet50_cifar10"
  dataset_type: "augmented"

# ── Arbitrary extra metadata ─────────────────────
extra:
  experiment_group: "ablation_study"
```

## Configuration precedence (highest to lowest)

1. **Runtime Python code** — `@step(enable_cache=False)` or `.with_options(enable_cache=False)`
2. **Step-level YAML** — `steps.train_model.enable_cache: false`
3. **Pipeline-level YAML** — `enable_cache: false`
4. **Default values** — in function signatures

## Settings key format

Settings are addressed as `<category>` or `<category>.<flavor>`:

| Key | Maps to |
|-----|---------|
| `docker` | `DockerSettings` |
| `resources` | `ResourceSettings` |
| `orchestrator.kubeflow` | Kubeflow orchestrator settings |
| `orchestrator.kubernetes` | Kubernetes orchestrator settings |
| `experiment_tracker.mlflow` | MLflow tracker settings |
| `step_operator.sagemaker` | SageMaker step operator settings |

## Multi-environment pattern

```
configs/
  dev.yaml       # Small dataset, no caching, no GPU
  staging.yaml   # Medium dataset, caching on
  prod.yaml      # Full dataset, GPU resources, caching on
```

```python
import os

env = os.getenv("ZENML_ENV", "dev")
my_pipeline.with_options(config_path=f"configs/{env}.yaml")()
```

### Example: dev.yaml

```yaml
enable_cache: false
parameters:
  dataset_path: "data/small.csv"
  learning_rate: 0.05
```

### Example: prod.yaml

```yaml
enable_cache: true
parameters:
  dataset_path: "s3://my-bucket/data/full.csv"
  learning_rate: 0.01
settings:
  resources:
    cpu_count: 8
    gpu_count: 1
    memory: "32GB"
```

## `with_options()` vs `configure()`

| Method | Mutates original? | Recommended |
|--------|:-----------------:|:-----------:|
| `with_options()` | No (returns copy) | Yes |
| `configure()` | Yes (in place) | Only for one-off scripts |

Always prefer `with_options()` inside pipeline definitions to avoid mutation side effects.
