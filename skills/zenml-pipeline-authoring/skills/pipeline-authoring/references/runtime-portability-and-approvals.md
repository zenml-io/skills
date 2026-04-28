# Runtime Portability and Approval Reference

Use this reference when a pipeline works locally but fails during remote packaging, container builds, Kubernetes execution, or production promotion.

## uv, pyproject, and package discovery

Remote orchestrators only get code and files that ZenML can package into the execution image. Treat `pyproject.toml` as the contract for what exists inside the container.

### Checklist

- Add `__init__.py` files to importable source folders such as `steps/`, `pipelines/`, `materializers/`, and `utils/`.
- Prefer editable local installs in development inside an activated environment: `uv pip install -e ".[dev]"`.
- In remote Docker settings, point ZenML at the project metadata with `pyproject_path="pyproject.toml"` when needed.
- If your project must be importable as a package inside the image, set `local_project_install_command`. If using `uv` inside the Docker image, target the container Python environment explicitly, for example `"uv pip install --system -e ."` or `"uv pip install --system -e '.[train]'"`, assuming `uv` is available in the image. Otherwise use `"pip install -e ."`.
- Do not depend on files that are only present on your laptop. Include templates, SQL files, tokenizer files, and small static assets as package data or load them from the artifact store / object storage.
- If using a private package or wheel, publish a Linux-compatible wheel to a package index or direct URL, or use a custom `dockerfile` plus `build_context_root` so the Docker build can copy/install that wheel. A wheel built on macOS, especially Apple Silicon, is usually not valid for Linux Kubernetes pods.

### Minimal pyproject package-discovery pattern

```toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "my-pipeline-project"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["zenml>=0.93", "pandas>=2.0"]

[tool.setuptools.packages.find]
where = ["."]
include = ["steps*", "pipelines*", "materializers*", "utils*"]

[tool.setuptools.package-data]
steps = ["*.sql", "*.html"]
utils = ["templates/*.html"]
```

If a step opens a file with `Path(__file__).parent / "template.html"`, make sure that file is included in the package or move it to a remote location and pass its URI as a parameter. Prefer `importlib.resources` for loading packaged files instead of relying on fragile relative paths.

## Apple Silicon development targeting AMD64 Linux/Kubernetes

Apple Silicon machines are ARM64. Many Kubernetes clusters are AMD64 Linux. The most common failure story is: local install succeeds on ARM macOS, then the remote pod tries to install or import an AMD64 Linux package that was never actually tested.

### Safe patterns

- Prefer packages that publish Linux AMD64 wheels (`manylinux_x86_64`) for your Python version.
- Avoid copying local `.venv`, local build artifacts, or macOS/ARM wheels into the remote image.
- If you maintain internal packages, build and publish Linux AMD64 wheels in CI, not on your laptop.
- Use a Linux AMD64 parent image when manually building custom base images for an AMD64 cluster.
- When building custom images from Apple Silicon for an AMD64 cluster, build in CI or use an explicit target platform such as `docker buildx build --platform linux/amd64 ...`.
- For native dependencies, list OS packages in `DockerSettings(apt_packages=[...])` instead of relying on what happens to be installed locally.
- If a package compiles during image build, add the compiler/system libraries it needs or prebuild a compatible wheel.

### Quick debugging questions

- Is the failing pod running on `linux/amd64` while development happened on `darwin/arm64`?
- Does the failing dependency have a wheel for the target Python version and platform?
- Did a local path dependency or wheel sneak into `requirements`/`pyproject.toml`, or require a custom `dockerfile` + `build_context_root` to be copied into the image?
- Are you using a base image that supports the cluster architecture?

## Kubernetes orchestrator troubleshooting

Kubernetes failures often happen before your Python code starts. Read the pod status and events first; they tell the story of what Kubernetes rejected or could not schedule.

### First commands to run

```bash
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --all-containers=true
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

### Common symptoms

| Symptom | Likely cause | Fix |
|---|---|---|
| `Forbidden`, `violates PodSecurity`, or admission webhook denial | Cluster admission policy rejects the pod spec | Adjust security context/image/user settings, namespace policy, service account, or ask the platform team which fields are allowed. |
| Pod stays `Pending` | Scheduler cannot place it | Lower or correct CPU/memory/GPU requests, choose the right node pool, or add required tolerations/node selectors in orchestrator settings. |
| `OOMKilled` | Container exceeded memory limit | Increase `ResourceSettings(memory=...)`, reduce batch size, or stream/chunk data. |
| `ImagePullBackOff` | Image registry/auth/tag problem | Check the image tag, registry credentials, service connector/secret, and whether the image was pushed. |
| `CrashLoopBackOff` after Python starts | Application/runtime error | Inspect logs, then reproduce with the same image and config if possible. |

### Resource requests and limits

Be explicit about resources for remote steps. A local laptop can opportunistically use memory and CPU; Kubernetes schedules from declared requests/limits.

```python
from zenml import step
from zenml.config import ResourceSettings

@step(settings={"resources": ResourceSettings(cpu_count=2, memory="8GB")})
def preprocess_step(...) -> ...:
    ...
```

Use step-level resource settings for heavy train/preprocess steps rather than making every step oversized.

## Human approval and model promotion

Separate two decisions that are often mixed together:

1. **Did the pipeline produce a candidate model/artifact?** The pipeline can answer this automatically with metrics and metadata.
2. **Should a human or release process promote it?** That can happen outside the pipeline through CI/CD approval gates, or through an approval checkpoint when your orchestrator / ZenML deployment supports a documented pause/resume or manual approval mechanism.

### CI/CD approval gate pattern

Use this when production promotion should be audited and controlled by the release system.

1. Training pipeline writes metrics, reports, and a model version to the ZenML Model Control Plane.
2. CI/CD reads the candidate run/model metadata and presents it for review.
3. A protected environment/manual approval step gates promotion.
4. After approval, the CI/CD job promotes the model version, tags artifacts, or triggers the deployment pipeline.

This keeps the pipeline deterministic: it creates evidence and candidates; the release system decides whether they become production.

### Approval checkpoint pattern

Use this only when your orchestrator or ZenML deployment supports a documented pause/resume or manual approval mechanism. The mental model is a checkpoint: the pipeline reaches an approval point, waits, then resumes once the required decision is supplied. If that mechanism is not available, prefer splitting the workflow into two pipelines: one pipeline produces the candidate model and evidence, and a separate deployment pipeline runs after external approval.

Good fits:

- Human review of a generated report before an expensive downstream step.
- Manual sign-off before pushing a model to a staging or production stage.
- Semi-automated workflows where rejection should stop the run cleanly.

Keep the approval payload small and explicit: candidate model name/version, key metrics, report URI, approver, decision, and timestamp. Avoid embedding large review artifacts directly in parameters; store them as artifacts or external reports and pass references.

### Promotion safety rules

- Do not silently promote to production just because evaluation passed a threshold; make the promotion policy explicit.
- Log the metrics and report URI that justified promotion.
- Use model versions/stages rather than overwriting a generic `latest` artifact.
- Prefer a separate deployment pipeline that consumes the approved model version.
- For CI/CD stage promotion, use `force=False` by default and handle occupied-stage errors; use `force=True` only when approval explicitly accepts archiving/replacing the currently staged model version.
