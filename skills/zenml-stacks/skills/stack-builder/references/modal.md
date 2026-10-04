# Modal Stack

A Modal stack runs the orchestration and each step in Modal sandboxes. GPU steps use the Modal step operator, which takes the GPU type from `ModalStepOperatorSettings(gpu=...)` and the count from `ResourceSettings(gpu_count=...)`.

Docs: [Modal orchestrator](https://docs.zenml.io/stacks/orchestrators/modal), [Modal step operator](https://docs.zenml.io/stacks/step-operators/modal).

## What the Stack Contains

| Component | Flavor | Why it is needed |
|-----------|--------|------------------|
| Orchestrator | `modal` | Runs the pipeline controller in a Modal sandbox |
| Step operator | `modal` | Runs GPU steps in their own Modal sandbox |
| Artifact store | `s3` | Modal cannot read a local artifact store; the stack validator rejects it |
| Container registry | `aws` (ECR) | Modal pulls the step image from a registry; a local registry is rejected |
| Image builder | `local` | The Modal orchestrator and step operator both require an image builder; it builds on the submitting machine, so Docker must run there |

Modal is not tied to one cloud: this reference uses S3 and ECR, but any remote artifact store and registry pair works. Swap the AWS sections for GCS and Artifact Registry, or Azure Blob and ACR, if that is what the user has.

## Two Kinds of Credentials

- **Modal credentials** let ZenML ask Modal to run work. Store them in a ZenML secret and reference them from the orchestrator and step operator. If neither `token_id` nor `token_secret` is set, Modal falls back to `MODAL_TOKEN_ID`/`MODAL_TOKEN_SECRET` or `~/.modal.toml`, which only works on machines that have them. Set both or neither: setting one fails validation.
- **Cloud credentials** let ZenML read and write the bucket and push and pull images. They live in an AWS service connector attached to the artifact store and registry. When a step starts, ZenML hands the registry credentials to Modal so it can pull the private image. Without them Modal tries an anonymous pull and the run fails with an authentication error, even though the stack registered fine.

## Step 1: Cloud Resources

Set shell variables once (none of them are secret):

```shell
export STACK_NAME=gpu-modal
export AWS_REGION=$(aws configure get region)
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME=${STACK_NAME}-artifacts-$(openssl rand -hex 3)
export TAG_SPEC="Key=zenml-stack,Value=${STACK_NAME}"
```

Create only what the Phase 2 inventory found missing.

**S3 bucket.** Outside `us-east-1`, S3 requires a location constraint:

```shell
aws s3api create-bucket --bucket "$BUCKET_NAME" --region "$AWS_REGION" \
  --create-bucket-configuration LocationConstraint="$AWS_REGION"   # omit this line in us-east-1
aws s3api put-bucket-tagging --bucket "$BUCKET_NAME" \
  --tagging "TagSet=[{Key=zenml-stack,Value=${STACK_NAME}}]"
```

**ECR repository.** ZenML pushes images as `<registry-uri>/zenml:<pipeline>`, so the repository must be called `zenml`. If it already exists, reuse it and do not tag it as yours.

```shell
aws ecr create-repository --repository-name zenml --region "$AWS_REGION" --tags "$TAG_SPEC"
```

**IAM role** with least-privilege S3 and ECR access, adapted from the [AWS stack guide](https://docs.zenml.io/stacks/popular-stacks/aws-guide). Write the policy files to a temporary directory, not the user's repository:

```shell
WORKDIR=$(mktemp -d)
cat > "$WORKDIR/trust.json" <<EOF
{"Version": "2012-10-17", "Statement": [{"Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::${AWS_ACCOUNT_ID}:root"},
  "Action": "sts:AssumeRole"}]}
EOF
cat > "$WORKDIR/access.json" <<EOF
{"Version": "2012-10-17", "Statement": [
  {"Effect": "Allow", "Action": ["s3:ListBucket", "s3:GetObject", "s3:PutObject",
    "s3:DeleteObject", "s3:GetBucketVersioning", "s3:ListBucketVersions",
    "s3:DeleteObjectVersion"],
   "Resource": ["arn:aws:s3:::${BUCKET_NAME}", "arn:aws:s3:::${BUCKET_NAME}/*"]},
  {"Effect": "Allow", "Action": "s3:ListAllMyBuckets", "Resource": "*"},
  {"Effect": "Allow", "Action": ["ecr:BatchGetImage", "ecr:BatchCheckLayerAvailability",
    "ecr:GetDownloadUrlForLayer", "ecr:GetAuthorizationToken", "ecr:InitiateLayerUpload",
    "ecr:UploadLayerPart", "ecr:CompleteLayerUpload", "ecr:PutImage",
    "ecr:DescribeRepositories", "ecr:ListRepositories", "ecr:DescribeImages"],
   "Resource": "*"}]}
EOF
aws iam create-role --role-name "${STACK_NAME}-zenml" \
  --assume-role-policy-document "file://$WORKDIR/trust.json" --tags "$TAG_SPEC"
aws iam put-role-policy --role-name "${STACK_NAME}-zenml" \
  --policy-name "${STACK_NAME}-zenml-access" --policy-document "file://$WORKDIR/access.json"
rm -rf "$WORKDIR"
```

An inline policy keeps the role self-contained, so teardown removes it with the role.

## Step 2: AWS Service Connector

Give ZenML the role through an AWS CLI profile, so no key is typed on a command line. The profile assumes the new role using the user's existing credentials. It adds a block to `~/.aws/config`, so include it in the Phase 3 plan:

```shell
aws configure set role_arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/${STACK_NAME}-zenml" --profile "${STACK_NAME}-zenml"
aws configure set source_profile "${AWS_PROFILE:-default}" --profile "${STACK_NAME}-zenml"
aws configure set region "$AWS_REGION" --profile "${STACK_NAME}-zenml"

AWS_PROFILE="${STACK_NAME}-zenml" zenml service-connector register "${STACK_NAME}-aws" \
  --type aws --auto-configure -l zenml-stack="$STACK_NAME"
zenml service-connector verify "${STACK_NAME}-aws"
```

`--auto-configure` picks up the role from the profile and registers an IAM-role connector. The role's default session lasts one hour, which is enough for a smoke test; see the [AWS service connector docs](https://docs.zenml.io/how-to/infrastructure-deployment/auth-management/aws-service-connector) for longer sessions or other authentication methods. If the user already has a verified connector that reaches the bucket and registry, reuse it and skip this step.

## Step 3: Modal Token Secret

Store the token pair as a ZenML secret without printing it or putting it on a command line. Read it from the environment (or ask the user to export it), write a private temporary file, and pass the file with `--values=@`:

```shell
umask 077
SECRET_FILE=$(mktemp)
python3 -c 'import json, os; print(json.dumps({"token_id": os.environ["MODAL_TOKEN_ID"], "token_secret": os.environ["MODAL_TOKEN_SECRET"]}))' > "$SECRET_FILE"
zenml secret create "${STACK_NAME}-modal" --values=@"$SECRET_FILE"
rm -f "$SECRET_FILE"
```

If the token only exists in `~/.modal.toml`, ask the user to run `zenml secret create <name> -i` themselves instead of reading the file.

## Step 4: Components and Stack

```shell
zenml artifact-store register "${STACK_NAME}-artifacts" -f s3 \
  --path="s3://${BUCKET_NAME}" --connector "${STACK_NAME}-aws" -l zenml-stack="$STACK_NAME"

zenml container-registry register "${STACK_NAME}-registry" -f aws \
  --uri="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com" \
  --connector "${STACK_NAME}-aws" -l zenml-stack="$STACK_NAME"

zenml image-builder register "${STACK_NAME}-builder" -f local \
  --use_subprocess_call=True -l zenml-stack="$STACK_NAME"

zenml orchestrator register "${STACK_NAME}-orchestrator" -f modal \
  --token_id="{{${STACK_NAME}-modal.token_id}}" \
  --token_secret="{{${STACK_NAME}-modal.token_secret}}" \
  -l zenml-stack="$STACK_NAME"

zenml step-operator register "${STACK_NAME}-gpu" -f modal \
  --token_id="{{${STACK_NAME}-modal.token_id}}" \
  --token_secret="{{${STACK_NAME}-modal.token_secret}}" \
  -l zenml-stack="$STACK_NAME"

zenml stack register "$STACK_NAME" \
  -o "${STACK_NAME}-orchestrator" -a "${STACK_NAME}-artifacts" \
  -c "${STACK_NAME}-registry" -i "${STACK_NAME}-builder" -s "${STACK_NAME}-gpu"
```

The `{{<secret>.<key>}}` values are [secret references](https://docs.zenml.io/how-to/secrets/secrets): ZenML stores the reference, not the token. The local image builder uses the Docker CLI (`use_subprocess_call=True`); also set `export DOCKER_BUILDKIT=1` before submitting runs.

## Modal-Specific Gotchas

- **Apple Silicon.** Modal only imports `linux/amd64` images. On an arm64 machine, build for `linux/amd64` with `DockerSettings` as shown in the [Modal step operator docs](https://docs.zenml.io/stacks/step-operators/modal); the smoke test in [smoke-test.md](smoke-test.md) already does this.
- **GPU type is required with a GPU count.** `ResourceSettings(gpu_count=1)` without `ModalStepOperatorSettings(gpu="...")` fails validation. A GPU type with no count means one GPU.
- **Region and cloud placement** (`region=`, `cloud=`) only work on Modal Team and Enterprise plans.
- **Timeouts.** The sandbox `timeout` defaults to 24 hours. Set a short one for the smoke test so a stuck step does not keep billing.
- **Schedules.** The Modal orchestrator does not support schedules. Use a different orchestrator, or a scheduler outside ZenML, if the user needs them.
