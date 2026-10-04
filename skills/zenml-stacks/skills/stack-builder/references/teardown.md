# Teardown

Remove only what this skill created for one stack. Anything without the `zenml-stack=<STACK_NAME>` label or tag stays, even if it looks unused.

## 1. Build the Removal List

List the ZenML objects that carry the label:

```python
from zenml.client import Client
from zenml.enums import StackComponentType

STACK_NAME = "gpu-modal"
client = Client()

def ours(model) -> bool:
    return (model.labels or {}).get("zenml-stack") == STACK_NAME

for component_type in StackComponentType:
    for component in client.list_stack_components(type=component_type, size=1000).items:
        if ours(component):
            print(component_type.value, component.name)
for connector in client.list_service_connectors(size=1000).items:
    if ours(connector):
        print("service_connector", connector.name)
```

And the AWS resources with the tag:

```shell
aws resourcegroupstaggingapi get-resources --tag-filters Key=zenml-stack,Values=<STACK_NAME> \
  --query "ResourceTagMappingList[].ResourceARN"
aws iam get-role --role-name <STACK_NAME>-zenml --query "Role.Tags"
aws eks list-access-entries --cluster-name <CLUSTER_NAME>   # Kubernetes only
```

IAM roles do not show up in the tagging API, so check the role on its own.

Show the user the full list, mark anything that holds data (the S3 bucket, ECR images), and ask for an explicit yes. Offer to keep the bucket and repository while removing everything else.

## 2. Remove in Reverse Order

```shell
zenml stack delete <STACK_NAME> -y            # switch away first if it is the active stack

zenml step-operator delete <STACK_NAME>-gpu
zenml orchestrator delete <STACK_NAME>-orchestrator
zenml image-builder delete <STACK_NAME>-builder
zenml container-registry delete <STACK_NAME>-registry
zenml artifact-store delete <STACK_NAME>-artifacts
zenml service-connector delete <STACK_NAME>-aws
zenml secret delete <STACK_NAME>-modal -y      # Modal only
```

Then the cloud side, only for items the list in step 1 showed as tagged:

```shell
# Kubernetes only: the access entry and the namespace, if this skill created them
aws eks delete-access-entry --cluster-name <CLUSTER_NAME> \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:role/<STACK_NAME>-zenml
kubectl get namespace zenml --show-labels    # delete only if labeled zenml-stack=<STACK_NAME>

aws iam delete-role-policy --role-name <STACK_NAME>-zenml --policy-name <STACK_NAME>-zenml-access
aws iam delete-role --role-name <STACK_NAME>-zenml

# Data-bearing resources: only with a separate yes
aws ecr delete-repository --repository-name zenml --force   # only if this skill created it
aws s3 rb s3://<BUCKET_NAME> --force
```

The `<STACK_NAME>-zenml` AWS CLI profile now points at a deleted role. Tell the user to remove its block from `~/.aws/config`; do not edit that file yourself.

`zenml stack delete` removes the stack record, not its components, so delete the components explicitly. If a component is still used by another stack, ZenML refuses to delete it: leave it and report it.

## 3. Confirm

Rerun step 1. Both lists should be empty. Report what was removed and anything kept on purpose.
