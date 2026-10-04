---
name: zenml-stack-builder
description: >-
  Build a working remote ZenML stack end to end: check what already exists,
  create the missing cloud resources (S3 bucket, ECR repository, IAM role,
  service connector, secrets), register the stack components and the stack,
  and prove it with a GPU smoke-test pipeline. Covers Modal (orchestrator and
  step operator with S3 + ECR) and Kubernetes on an existing EKS cluster
  (orchestrator and step operator with S3 + ECR). Use this skill whenever the
  user asks to set up, create, provision, or register a ZenML stack, wants to
  run ZenML pipelines or single steps on Modal, Kubernetes, EKS, or GPUs, asks
  "how do I get my pipeline off my laptop", needs a step operator for
  training, or has a half-configured stack that fails with registry, artifact
  store, or GPU scheduling errors. Also use it when an autonomous agent needs
  compute before it can run a training pipeline. If the user only asks a
  conceptual question about stacks ("what is a step operator?"), answer it
  directly and link the docs -- no need to run the workflow.
---

# Build a ZenML Stack

The ZenML docs show how to *register* a stack on infrastructure you already have (see [Popular Stacks](https://docs.zenml.io/stacks/popular-stacks/aws-guide)). This skill goes further: it finds what is missing, creates it with the user's approval, registers everything, and runs a pipeline on the new stack to prove it works. A stack is done when the smoke test passes, not when `zenml stack register` returns.

| Target | Orchestrator | GPU steps | Storage and images | Reference |
|--------|--------------|-----------|--------------------|-----------|
| **Modal** | `modal` | `modal` step operator | S3 artifact store, ECR registry, local image builder | [references/modal.md](references/modal.md) |
| **Kubernetes on EKS** | `kubernetes` | `kubernetes` step operator | S3 artifact store, ECR registry, local image builder | [references/aws-kubernetes.md](references/aws-kubernetes.md) |

For other targets (SageMaker, Vertex AI, AzureML), follow the matching [Popular Stacks](https://docs.zenml.io/stacks/popular-stacks/aws-guide) guide and reuse this skill's phases, approval gate, and smoke test.

## Ground Rules

These apply to every phase. Do not skip them, even when the user is in a hurry.

1. **One approval before anything that costs money or creates cloud resources.** Show the plan from Phase 3 and wait for an explicit yes. A yes covers that plan only; a changed plan needs a new yes.
2. **Never modify or delete what you did not create.** Reuse existing buckets, repositories, roles, connectors, and components as they are. If something existing is misconfigured, report it and ask; do not fix it silently.
3. **Label everything you create.** Tag AWS resources with `zenml-stack=<STACK_NAME>` and register ZenML components and service connectors with `-l zenml-stack=<STACK_NAME>`. Teardown depends on these labels.
4. **Keep credentials out of files, flags, and logs.** Store Modal tokens in a ZenML secret and reference them as `{{<SECRET_NAME>.<KEY>}}`. Give AWS access to ZenML through a service connector. Never commit credentials, echo them, or pass them as plain command-line values that land in shell history.
5. **Use stable names.** Derive every name from the stack name (`<STACK_NAME>-artifacts`, `<STACK_NAME>-registry`, and so on) so a rerun finds what the last run created instead of creating duplicates.
6. **Work the same way with or without a person at the terminal.** If you run headless (for example as an autonomous agent), send the Phase 3 plan through whatever approval channel you have and stop until it is answered. Never treat silence as a yes.

## Phase 0: Ask What the User Needs

Infer answers from context when they are obvious; ask only when they are not.

- **Q1: Which target?** Modal for fast, per-step GPU access without running a cluster. Kubernetes when the team already runs an EKS cluster with GPU nodes.
- **Q2: Stack name and region?** Default the region to the AWS CLI's configured region.
- **Q3: Which GPU, and how many per step?** For Modal this is a GPU type string (for example `A100`, `H100`, `B200`; check [Modal's GPU list](https://modal.com/docs/guide/gpu) for current names). For Kubernetes it is whatever the cluster's GPU node group provides.
- **Q4: Reuse or create?** Ask whether existing buckets, registries, or connectors should be reused. The inventory in Phase 2 will show candidates.
- **Q5: Make it the active stack afterwards?** Default to no.

## Phase 1: Preflight

Run these checks and stop with a clear message if one fails. Each check names what to fix.

| Check | Command | Must show |
|-------|---------|-----------|
| ZenML client | `zenml version` | A recent ZenML release |
| Remote server | `zenml status` | Connected to a deployed ZenML server, not a local database. Modal sandboxes and Kubernetes pods must reach the server. |
| Integrations | `zenml integration install aws s3 modal -y` (Modal) or `zenml integration install aws s3 kubernetes -y` (Kubernetes) | Installs cleanly |
| Docker | `docker info` | The daemon is running. The local image builder needs it. |
| AWS identity | `aws sts get-caller-identity` | The intended account. Note the account ID. |
| AWS region | `aws configure get region` | The region from Q2 |
| Modal auth (Modal only) | `MODAL_TOKEN_ID` and `MODAL_TOKEN_SECRET` set, or a `~/.modal.toml` profile | A token pair exists. Do not print it. |
| Cluster access (Kubernetes only) | `kubectl config current-context` and `kubectl get nodes -o custom-columns=NAME:.metadata.name,GPUS:.status.allocatable.nvidia\.com/gpu` | The EKS cluster, with at least one GPU node |

If the ZenML server is local, stop: remote compute cannot call back to it. Point the user to [deploying ZenML](https://docs.zenml.io/deploying-zenml/deploying-zenml) or ZenML Pro.

## Phase 2: Inventory What Exists

Build a list of candidates before planning anything:

```shell
zenml stack list
zenml artifact-store list
zenml container-registry list
zenml orchestrator list
zenml step-operator list
zenml image-builder list
zenml service-connector list
zenml service-connector list-resources --resource-type s3-bucket
zenml service-connector list-resources --resource-type docker-registry
zenml service-connector list-resources --resource-type kubernetes-cluster   # Kubernetes only
aws s3api list-buckets --query "Buckets[].Name"
aws ecr describe-repositories --query "repositories[].repositoryName"
```

Look for a stack or components that already carry the `zenml-stack=<STACK_NAME>` label. Those are from an earlier run of this skill and can be reused.

## Phase 3: Plan and Ask for Approval

Present one table and one question. Example:

```text
Plan for stack "gpu-modal" in eu-central-1 (account 123456789012)

| Piece              | Status   | Action | Name                         | Cost                          |
|--------------------|----------|--------|------------------------------|-------------------------------|
| S3 bucket          | missing  | create | gpu-modal-artifacts-1a2b     | storage and requests only     |
| ECR repository     | exists   | reuse  | zenml                        | none                          |
| IAM role           | missing  | create | gpu-modal-zenml              | none                          |
| AWS connector      | missing  | create | gpu-modal-aws                | none                          |
| Modal token secret | missing  | create | gpu-modal-modal              | none                          |
| Components + stack | missing  | create | gpu-modal-*                  | none                          |
| Smoke test         | -        | run    | one step on 1x H100          | Modal GPU time, a few minutes |

Go ahead? (yes / change something / stop)
```

Do not create anything until the answer is an explicit yes.

## Phase 4: Create the Missing Pieces

Follow the target's reference file for exact commands, in this order:

1. **Cloud resources**: S3 bucket, ECR repository, IAM role and least-privilege policies. Tag each with `zenml-stack=<STACK_NAME>`.
2. **Secrets**: the Modal token secret (Modal only).
3. **Service connector**: one AWS connector for the bucket, the registry, and (for Kubernetes) the EKS cluster. Run `zenml service-connector verify <CONNECTOR>` before using it.
4. **Components**: artifact store, container registry, image builder, orchestrator, step operator. Connect each AWS-backed component to the connector.

After each step, record what was created (type, name, ID or ARN) in a running list. You need it for the report and for teardown.

If a command fails halfway, stop, show the error and what has been created so far, and ask how to proceed. Do not retry destructive or billable commands in a loop.

## Phase 5: Register the Stack

```shell
zenml stack register <STACK_NAME> \
  -o <ORCHESTRATOR> -a <ARTIFACT_STORE> -c <CONTAINER_REGISTRY> \
  -i <IMAGE_BUILDER> -s <STEP_OPERATOR>
zenml stack describe <STACK_NAME>
```

Add `--set` only if the user said yes to Q5. Check that `zenml stack describe` lists every component you expect.

## Phase 6: Prove It with a Smoke Test

Run the one-step GPU pipeline from [references/smoke-test.md](references/smoke-test.md) on the new stack. The stack passes when:

- the run finishes with status `completed`;
- the `gpu_report` artifact loads back with `Client().get_artifact_version(...)` and shows the requested GPU model;
- the step ran on the step operator (the step's run metadata or logs show the Modal sandbox or Kubernetes pod).

If it fails, use the troubleshooting table below, fix the cause, and rerun. Count the stack as built only after a passing run.

## Phase 7: Report

End with a short report:

```text
Stack "<STACK_NAME>" is ready. Smoke test: run <RUN_ID> completed on <GPU>.

Created:  <type> <name> (<id or ARN>), one per line
Reused:   <type> <name>, one per line
Use it:   zenml stack set <STACK_NAME>, then @step(step_operator=True) on GPU steps
Remove:   see references/teardown.md (deletes only items labeled zenml-stack=<STACK_NAME>)
```

Then suggest next steps: `zenml-pipeline-authoring` to write pipelines that use the step operator, and `zenml-quick-wins` for production hardening.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Modal fails to pull the image with an authentication error | The registry has no credentials, so Modal tried an anonymous pull | Connect the container registry to the AWS connector |
| Modal rejects the image during import | The image was built for `arm64` on Apple Silicon | Build `linux/amd64` images (see [references/modal.md](references/modal.md)) |
| Stack validation rejects a local artifact store or registry | Modal and Kubernetes pods cannot read local paths | Use the S3 artifact store and ECR registry |
| `GPU resources requested ... but no GPU type was specified` | `ResourceSettings(gpu_count>0)` without `ModalStepOperatorSettings(gpu=...)` | Set the GPU type in the step operator settings |
| Kubernetes pod stays `Pending` with insufficient `nvidia.com/gpu` | No GPU node can take the pod, the NVIDIA device plugin is missing, or a taint blocks it | Check `kubectl describe pod`, add the toleration and node selector from [references/smoke-test.md](references/smoke-test.md), or install the [NVIDIA device plugin](https://github.com/NVIDIA/k8s-device-plugin) |
| `Unauthorized` from the EKS cluster through the connector | The connector's IAM principal is not mapped into the cluster | Add an EKS access entry or `aws-auth` mapping (see [references/aws-kubernetes.md](references/aws-kubernetes.md)) |
| Steps cannot reach the ZenML server | The server is local or not reachable from the remote compute | Use a deployed ZenML server |

## Tearing Down

When the user wants the stack gone, follow [references/teardown.md](references/teardown.md). It deletes only what carries the `zenml-stack=<STACK_NAME>` label, in reverse order of creation, and asks before deleting anything that holds data.

## Related Docs

- [Stacks and components](https://docs.zenml.io/stacks)
- [Service connectors guide](https://docs.zenml.io/how-to/infrastructure-deployment/auth-management/service-connectors-guide)
- [Secrets](https://docs.zenml.io/how-to/secrets/secrets)
- [Modal step operator](https://docs.zenml.io/stacks/step-operators/modal) and [Modal orchestrator](https://docs.zenml.io/stacks/orchestrators/modal)
- [Kubernetes step operator](https://docs.zenml.io/stacks/step-operators/kubernetes) and [Kubernetes orchestrator](https://docs.zenml.io/stacks/orchestrators/kubernetes)
