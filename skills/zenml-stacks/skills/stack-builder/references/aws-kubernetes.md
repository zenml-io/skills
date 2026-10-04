# Kubernetes Stack on EKS

This stack runs pipelines as pods on an **existing** EKS cluster. GPU steps go through the Kubernetes step operator, which turns `ResourceSettings(gpu_count=N)` into `nvidia.com/gpu` requests and limits on the step pod.

The skill does not create the EKS cluster or its GPU node group. That is a larger, slower, and more expensive change that usually belongs to whoever runs the cluster. If there is no cluster yet, stop after Phase 1, tell the user what is missing, and point them to the [EKS user guide](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html) and the [NVIDIA device plugin](https://github.com/NVIDIA/k8s-device-plugin).

Docs: [Kubernetes stack guide](https://docs.zenml.io/stacks/popular-stacks/kubernetes), [Kubernetes orchestrator](https://docs.zenml.io/stacks/orchestrators/kubernetes), [Kubernetes step operator](https://docs.zenml.io/stacks/step-operators/kubernetes), [AWS service connector](https://docs.zenml.io/how-to/infrastructure-deployment/auth-management/aws-service-connector).

## What the Stack Contains

| Component | Flavor | Why it is needed |
|-----------|--------|------------------|
| Orchestrator | `kubernetes` | Runs the pipeline as pods in the cluster |
| Step operator | `kubernetes` | Runs GPU steps as separate pods with their own resource requests |
| Artifact store | `s3` | Pods need remote storage |
| Container registry | `aws` (ECR) | Pods pull the step image from ECR |
| Image builder | `local` | Builds the image on the submitting machine with Docker |

## Before You Start

Check the cluster with the user's own `kubectl` access:

```shell
export STACK_NAME=gpu-eks
export CLUSTER_NAME=<EKS_CLUSTER_NAME>
export AWS_REGION=$(aws configure get region)
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export K8S_NAMESPACE=zenml
export TAG_SPEC="Key=zenml-stack,Value=${STACK_NAME}"

aws eks describe-cluster --name "$CLUSTER_NAME" --query "cluster.accessConfig.authenticationMode"
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPUS:.status.allocatable.nvidia\.com/gpu,TAINTS:.spec.taints
```

You need at least one node with allocatable `nvidia.com/gpu`. Note any GPU taints; the smoke test needs a matching toleration. The authentication mode decides how the connector's role gets into the cluster in Step 3.

## Step 1: Cloud Resources

The S3 bucket and ECR repository are the same as for Modal. Follow "Step 1: Cloud Resources" in [modal.md](modal.md) for the bucket, the `zenml` ECR repository, and the role, with one change: add EKS read access to `access.json` so the connector can find the cluster:

```json
{"Effect": "Allow", "Action": ["eks:ListClusters", "eks:DescribeCluster"],
 "Resource": "arn:aws:eks:${AWS_REGION}:${AWS_ACCOUNT_ID}:cluster/${CLUSTER_NAME}"}
```

`eks:ListClusters` is not resource-scoped in IAM; if the policy is rejected, move it to its own statement with `"Resource": "*"`.

## Step 2: Kubernetes Namespace

ZenML does not create the namespace; it does create its own service account and role binding inside it on the first run. Create the namespace only if it is missing:

```shell
kubectl get namespace "$K8S_NAMESPACE" || \
  { kubectl create namespace "$K8S_NAMESPACE" && \
    kubectl label namespace "$K8S_NAMESPACE" zenml-stack="$STACK_NAME"; }
```

## Step 3: Let the Role Into the Cluster

An IAM principal that did not create the cluster has no Kubernetes permissions until it is mapped in. This changes the access configuration of a cluster you did not create, so list it separately in the Phase 3 plan and get explicit approval.

If the authentication mode includes `API` (`API` or `API_AND_CONFIG_MAP`), add an access entry scoped to the namespace. ZenML needs to create pods, jobs, a service account, and a role binding there, so use the admin policy at namespace scope, not cluster scope:

```shell
ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${STACK_NAME}-zenml"
aws eks create-access-entry --cluster-name "$CLUSTER_NAME" --principal-arn "$ROLE_ARN" \
  --tags zenml-stack="$STACK_NAME"
aws eks associate-access-policy --cluster-name "$CLUSTER_NAME" --principal-arn "$ROLE_ARN" \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSAdminPolicy \
  --access-scope type=namespace,namespaces="$K8S_NAMESPACE"
```

If the mode is `CONFIG_MAP` only, the role has to be added to the `aws-auth` ConfigMap. Editing that ConfigMap can lock people out of the cluster, so do not do it yourself: give the cluster admin the role ARN and the [AWS instructions](https://docs.aws.amazon.com/eks/latest/userguide/auth-configmap.html), and wait.

## Step 4: AWS Service Connector

Same as "Step 2: AWS Service Connector" in [modal.md](modal.md). Then confirm the connector can reach all three resources:

```shell
zenml service-connector verify "${STACK_NAME}-aws" --resource-type kubernetes-cluster
zenml service-connector list-resources --connector-type aws --resource-type kubernetes-cluster
```

The cluster name must appear in the resource list. If verification fails with `Unauthorized`, Step 3 is not in effect yet.

## Step 5: Components and Stack

```shell
zenml artifact-store register "${STACK_NAME}-artifacts" -f s3 \
  --path="s3://${BUCKET_NAME}" --connector "${STACK_NAME}-aws" -l zenml-stack="$STACK_NAME"

zenml container-registry register "${STACK_NAME}-registry" -f aws \
  --uri="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com" \
  --connector "${STACK_NAME}-aws" -l zenml-stack="$STACK_NAME"

zenml image-builder register "${STACK_NAME}-builder" -f local -l zenml-stack="$STACK_NAME"

zenml orchestrator register "${STACK_NAME}-orchestrator" -f kubernetes \
  --kubernetes_namespace="$K8S_NAMESPACE" -l zenml-stack="$STACK_NAME"
zenml orchestrator connect "${STACK_NAME}-orchestrator" \
  --connector "${STACK_NAME}-aws" --resource-id "$CLUSTER_NAME"

zenml step-operator register "${STACK_NAME}-gpu" -f kubernetes \
  --kubernetes_namespace="$K8S_NAMESPACE" -l zenml-stack="$STACK_NAME"
zenml step-operator connect "${STACK_NAME}-gpu" \
  --connector "${STACK_NAME}-aws" --resource-id "$CLUSTER_NAME"

zenml stack register "$STACK_NAME" \
  -o "${STACK_NAME}-orchestrator" -a "${STACK_NAME}-artifacts" \
  -c "${STACK_NAME}-registry" -i "${STACK_NAME}-builder" -s "${STACK_NAME}-gpu"
```

Connecting through the service connector (rather than a local `kubernetes_context`) means the stack also works from machines that have no kubeconfig, including CI and autonomous agents.

## Kubernetes-Specific Gotchas

- **GPU pods stay `Pending`.** Check `kubectl -n "$K8S_NAMESPACE" describe pod <pod>`. Common causes: no node with free `nvidia.com/gpu`, the NVIDIA device plugin is not installed, or a GPU taint without a matching toleration. Set tolerations and node selectors through `KubernetesStepOperatorSettings(pod_settings=KubernetesPodSettings(...))`; [smoke-test.md](smoke-test.md) shows the shape.
- **Pods fail to pull the image.** EKS nodes pull from ECR with the node role. Make sure the node role can read the `zenml` repository (the managed `AmazonEC2ContainerRegistryReadOnly` policy is the usual way).
- **Steps cannot reach ZenML.** Pods call back to the ZenML server, so the server URL must be reachable from inside the cluster.
- **Debugging.** ZenML labels pods with `run` and `pipeline`, so `kubectl -n "$K8S_NAMESPACE" get pods -l pipeline=<name>` finds them.
