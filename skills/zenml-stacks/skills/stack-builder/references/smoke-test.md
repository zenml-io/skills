# GPU Smoke Test

A stack is built when a pipeline runs on it, not when it registers. This test runs one step on the step operator with one GPU, records what `nvidia-smi` sees, and reads the result back from the artifact store. That covers the whole path: image build and push, registry pull, remote execution, GPU scheduling, artifact write, and artifact read.

## The Pipeline

Write this to a scratch location (not the user's project) as `gpu_smoke_test.py`. Set `TARGET` to `modal` or `kubernetes` and adjust the GPU fields for the stack.

```python
"""One-step GPU smoke test for a new ZenML stack."""

import subprocess
from typing import Annotated

from zenml import pipeline, step
from zenml.config import DockerSettings, ResourceSettings
from zenml.config.docker_settings import DockerBuildConfig, DockerBuildOptions

TARGET = "modal"  # or "kubernetes"
MODAL_GPU_TYPE = "H100"  # any GPU type your Modal plan offers, e.g. "A100" or "B200"
K8S_GPU_TOLERATION_KEY = "nvidia.com/gpu"  # match the taint on your GPU nodes
K8S_GPU_NODE_SELECTOR: dict[str, str] = {}  # e.g. {"eks.amazonaws.com/nodegroup": "gpu"}


def gpu_step_settings() -> dict:
    """Build the step operator and resource settings for the target."""
    settings: dict = {"resources": ResourceSettings(gpu_count=1)}
    if TARGET == "modal":
        from zenml.integrations.modal.flavors import ModalStepOperatorSettings

        settings["step_operator"] = ModalStepOperatorSettings(
            gpu=MODAL_GPU_TYPE, timeout=900
        )
    else:
        from zenml.integrations.kubernetes.flavors import (
            KubernetesStepOperatorSettings,
        )
        from zenml.integrations.kubernetes.pod_settings import KubernetesPodSettings

        settings["step_operator"] = KubernetesStepOperatorSettings(
            pod_settings=KubernetesPodSettings(
                node_selectors=K8S_GPU_NODE_SELECTOR,
                tolerations=[
                    {
                        "key": K8S_GPU_TOLERATION_KEY,
                        "operator": "Exists",
                        "effect": "NoSchedule",
                    }
                ],
            ),
            active_deadline_seconds=900,
        )
    return settings


# Modal only imports linux/amd64 images; this also keeps Apple Silicon builds portable.
amd64 = DockerBuildConfig(build_options=DockerBuildOptions(platform="linux/amd64"))
docker_settings = DockerSettings(parent_image_build_config=amd64, build_config=amd64)


@step(step_operator=True, settings=gpu_step_settings())
def gpu_check() -> Annotated[str, "gpu_report"]:
    """Report the GPUs visible to this step."""
    result = subprocess.run(
        ["nvidia-smi", "--query-gpu=name,memory.total", "--format=csv,noheader"],
        capture_output=True,
        text=True,
        check=True,
    )
    return result.stdout.strip()


@pipeline(settings={"docker": docker_settings}, enable_cache=False)
def gpu_smoke_test() -> None:
    """Run one GPU step on the active stack's step operator."""
    gpu_check()


if __name__ == "__main__":
    run = gpu_smoke_test()
    if run is not None:
        report = run.steps["gpu_check"].outputs["gpu_report"][0].load()
        print(f"Run {run.id} finished with status {run.status}. GPU report: {report}")
```

The `nvidia-smi` binary is present on Modal GPU sandboxes and on EKS GPU nodes through the NVIDIA container runtime, so the default ZenML image is enough. Caching is off so a rerun really runs.

## Running It

```shell
zenml stack set <STACK_NAME>     # or: ZENML_ACTIVE_STACK_ID=<id> python gpu_smoke_test.py
export DOCKER_BUILDKIT=1
python gpu_smoke_test.py
```

If the user did not want the new stack made active, switch back afterwards with `zenml stack set <PREVIOUS_STACK>`.

## What Counts as a Pass

All three must hold:

1. The run's status is `completed`.
2. The `gpu_report` artifact loads and names the GPU you asked for (for example `NVIDIA H100 80GB HBM3, 81559 MiB`).
3. The step ran on the step operator: on Modal, the step run metadata includes a sandbox ID; on Kubernetes, `kubectl -n <namespace> get pods -l pipeline=gpu_smoke_test` shows the step pod.

Anything else is a failure. Report it with the step logs (`zenml` dashboard or `run.steps["gpu_check"]`) and use the troubleshooting table in `SKILL.md`.

## Cost

One step on one GPU for a few minutes, plus image build and push time on the submitting machine. The step sets a 15-minute ceiling (`timeout=900` on Modal, `active_deadline_seconds=900` on Kubernetes) so a stuck step stops billing.
