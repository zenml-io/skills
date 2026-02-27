# Docker Settings Reference

`DockerSettings` controls how ZenML builds container images for remote execution. When running on Kubernetes, Vertex AI, SageMaker, or AzureML, ZenML builds a Docker image containing your code and dependencies.

## Quick start

```python
from zenml.config import DockerSettings

docker = DockerSettings(
    requirements=["scikit-learn>=1.0", "pandas>=2.0"],
    apt_packages=["libgomp1"],
)

@pipeline(settings={"docker": docker})
def my_pipeline() -> None:
    ...
```

## Field catalog by use case

### Python packages

| Field | Type | Purpose |
|-------|------|---------|
| `requirements` | `str \| list[str]` | Pip packages or path to requirements.txt |
| `required_integrations` | `list[str]` | ZenML integrations to install (e.g., `["mlflow", "kubeflow"]`) |
| `install_stack_requirements` | `bool` (default: `True`) | Auto-install stack component deps |
| `replicate_local_python_environment` | `bool` | Run `pip freeze` and install everything locally installed |
| `pyproject_path` | `str` | Path to pyproject.toml for dependency export |
| `local_project_install_command` | `str` | Command to install local project (e.g., `"pip install -e .[train]"`) |

**Installation order**: local env → stack requirements → integrations → pyproject → explicit requirements.

### System packages

| Field | Type | Purpose |
|-------|------|---------|
| `apt_packages` | `list[str]` | APT packages to install (e.g., `["git", "libgomp1", "ffmpeg"]`) |

### Environment variables

| Field | Type | Purpose |
|-------|------|---------|
| `environment` | `dict[str, Any]` | Set *before* pip install (build-time). Use for build-affecting vars. |
| `runtime_environment` | `dict[str, Any]` | Set *after* pip install (runtime). Use for application config. |

**The distinction matters**: `environment` is available during the Docker build (affects package installation). `runtime_environment` is only available when the step runs. Both support `${VAR_NAME}` syntax to reference host environment variables.

```python
DockerSettings(
    environment={"PIP_INDEX_URL": "https://my-registry.com/simple"},     # Build-time
    runtime_environment={"API_KEY": "${MY_API_KEY}", "MODEL_VERSION": "v2"},  # Runtime
)
```

### Base images

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `parent_image` | `str` | ZenML base image | Custom base Docker image |
| `dockerfile` | `str` | None | Path to custom Dockerfile (overrides `parent_image`) |
| `build_context_root` | `str` | None | Build context for custom Dockerfile |
| `skip_build` | `bool` | `False` | Use `parent_image` directly, no build. Requires `parent_image`. |

### Build control

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `prevent_build_reuse` | `bool` | `False` | Force fresh build (useful for debugging stale images) |
| `target_repository` | `str` | None | Docker repo name (appended to container registry URI) |
| `image_tag` | `str` | None | Custom tag for the built image |
| `build_config` | `DockerBuildConfig` | None | Dockerignore path, build options (no_cache, etc.) |

### Package installer

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `python_package_installer` | `"uv" \| "pip"` | `"uv"` | Package installer. `uv` is faster and the default. |
| `python_package_installer_args` | `dict` | `{}` | Extra args passed to the installer |

### Code inclusion

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `allow_including_files_in_images` | `bool` | `True` | Include source code in images |
| `allow_download_from_code_repository` | `bool` | `True` | Download from registered code repo |
| `allow_download_from_artifact_store` | `bool` | `True` | Download from artifact store |

ZenML tries code download first (from code repository, then artifact store), then falls back to including in the image.

### Container user

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `user` | `str` | None | Run container as this UID/user. Sets ownership of `/app`. |

## Pipeline-level vs step-level

Pipeline-level applies to all steps:

```python
@pipeline(settings={"docker": docker_settings})
def my_pipeline() -> None: ...
```

Step-level overrides pipeline-level for that step:

```python
@step(settings={"docker": gpu_docker})
def train_step() -> None: ...
```

Or via `with_options()`:

```python
train_step.with_options(settings={"docker": gpu_docker})
```

## YAML equivalents

```yaml
# Pipeline-level
settings:
  docker:
    requirements:
      - scikit-learn>=1.0
      - pandas>=2.0
    required_integrations:
      - mlflow
    apt_packages:
      - libgomp1
    environment:
      PYTHONUNBUFFERED: "1"
    runtime_environment:
      API_KEY: ${MY_API_KEY}
    python_package_installer: uv

# Step-level override
steps:
  train_step:
    settings:
      docker:
        parent_image: pytorch/pytorch:2.2.0-cuda11.8-cudnn8-runtime
        requirements:
          - transformers>=4.0
```

## GPU and distributed training

When a step needs GPU resources, you need **both** a CUDA-enabled base image **and** resource requests.

### Requesting GPU resources

```python
from zenml import step
from zenml.config import ResourceSettings

@step(settings={
    "resources": ResourceSettings(cpu_count=8, gpu_count=2, memory="16GB")
})
def train_step(...):
    ...
```

Not all orchestrators support `ResourceSettings` — some (e.g., SkyPilot) expose dedicated settings. Check your orchestrator's docs.

### CUDA-enabled Docker image

Requesting a GPU is not enough — the Docker image needs the CUDA runtime too:

```python
from zenml.config import DockerSettings

docker = DockerSettings(
    parent_image="pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime",
    python_package_installer_args={"system": None},
    requirements=["zenml", "torchvision"],
)

@pipeline(settings={"docker": docker})
def gpu_pipeline() -> None:
    ...
```

Use official CUDA images from PyTorch, TensorFlow, or your cloud provider. The `python_package_installer_args={"system": None}` tells uv to install into the system Python (since CUDA base images often don't use virtual environments).

### Step-level GPU override

Apply GPU Docker settings only to the training step (other steps use the lighter pipeline-level image):

```python
gpu_docker = DockerSettings(
    parent_image="pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime",
    python_package_installer_args={"system": None},
    requirements=["zenml", "torchvision", "transformers"],
)

@step(settings={"docker": gpu_docker, "resources": ResourceSettings(gpu_count=1)})
def train_step() -> None:
    ...
```

### Multi-GPU with HuggingFace Accelerate

For distributed training across multiple GPUs or nodes:

```python
from zenml.integrations.huggingface.steps import run_with_accelerate

@run_with_accelerate(num_processes=4, multi_gpu=True)
@step
def distributed_train_step(...):
    ...  # Your distributed training code
```

Add `accelerate` to your Docker requirements. Accelerate-decorated steps must be called with keyword arguments.

### CUDA memory management

For memory-heavy GPU steps, clear the CUDA cache at the start:

```python
import gc, torch

@step
def gpu_step(...):
    gc.collect()
    torch.cuda.empty_cache()
    ...  # Your GPU code
```

---

## Common debugging patterns

**"My remote run still uses old code"**: Set `prevent_build_reuse=True` to force a fresh build.

**"ModuleNotFoundError in container"**: Add the missing package to `requirements` or `required_integrations`.

**"pip install fails in build"**: Check `environment` for `PIP_INDEX_URL` or `PIP_EXTRA_INDEX_URL` if using a private registry.

**"Container runs as wrong user"**: Set `user="1000"` to run as a specific UID.

**Reducing image size**: Use `.dockerignore` via `build_config=DockerBuildConfig(dockerignore=".dockerignore")`.
