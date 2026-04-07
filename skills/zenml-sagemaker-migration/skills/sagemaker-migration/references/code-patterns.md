# SageMaker Pipelines → ZenML Code Patterns

These examples show the most common SageMaker-to-ZenML translations. They are not meant to be copied blindly. Use them as patterns, then adjust imports, dependencies, and settings to match the user's actual pipeline.

## 1. Simple `ProcessingStep` + `TrainingStep` -> `@step` + `@pipeline`

### SageMaker SDK

```python
from sagemaker.processing import ScriptProcessor
from sagemaker.sklearn.estimator import SKLearn
from sagemaker.workflow.pipeline import Pipeline
from sagemaker.workflow.pipeline_context import PipelineSession
from sagemaker.workflow.steps import ProcessingStep, TrainingStep

pipeline_session = PipelineSession()

processor = ScriptProcessor(..., sagemaker_session=pipeline_session)
estimator = SKLearn(..., sagemaker_session=pipeline_session)

step_process = ProcessingStep(
    name="Preprocess",
    step_args=processor.run(code="preprocess.py"),
)

step_train = TrainingStep(
    name="Train",
    step_args=estimator.fit(inputs={"train": step_process.properties.ProcessingOutputConfig.Outputs["train"].S3Output.S3Uri}),
)

pipeline = Pipeline(name="train-pipeline", steps=[step_process, step_train])
```

### ZenML

```python
from typing import Tuple

from zenml import pipeline, step

@step
def preprocess() -> Tuple[str, str]:
    train_uri = "s3://bucket/processed/train"
    validation_uri = "s3://bucket/processed/validation"
    return train_uri, validation_uri

@step
def train(train_uri: str, validation_uri: str) -> str:
    return "s3://bucket/model/model.tar.gz"

@pipeline
def training_pipeline() -> None:
    train_uri, validation_uri = preprocess()
    train(train_uri=train_uri, validation_uri=validation_uri)
```

**Migration note:** the graph now expresses data flow through typed step outputs, not through step-property placeholders.

## 2. Pipeline parameters -> typed Python parameters

### SageMaker SDK

```python
from sagemaker.workflow.parameters import ParameterFloat, ParameterString

input_data = ParameterString(name="InputData", default_value="s3://bucket/raw.csv")
threshold = ParameterFloat(name="Threshold", default_value=0.9)
```

### ZenML

```python
from zenml import pipeline, step

@step
def evaluate(input_data_uri: str, threshold: float) -> bool:
    return threshold >= 0.9

@pipeline
def pipeline_with_params(input_data_uri: str = "s3://bucket/raw.csv", threshold: float = 0.9) -> None:
    evaluate(input_data_uri=input_data_uri, threshold=threshold)
```

**Migration warning:** if the original SageMaker parameter controlled infrastructure such as `instance_type` or `instance_count`, do **not** map it to an ordinary business-logic parameter. Move that concern into orchestrator settings instead.

## 3. `PropertyFile` + `JsonGet` -> artifacts and normal Python access

### SageMaker SDK

```python
from sagemaker.workflow.functions import JsonGet
from sagemaker.workflow.properties import PropertyFile
from sagemaker.workflow.steps import ProcessingStep

evaluation_report = PropertyFile(
    name="EvaluationReport",
    output_name="evaluation",
    path="evaluation.json",
)

step_eval = ProcessingStep(
    name="Evaluate",
    property_files=[evaluation_report],
    step_args=processor.run(code="evaluate.py"),
)

accuracy = JsonGet(
    step_name=step_eval.name,
    property_file=evaluation_report,
    json_path="metrics.accuracy",
)
```

### ZenML

```python
from zenml import pipeline, step

@step
def evaluate() -> dict[str, float]:
    return {"accuracy": 0.93, "f1": 0.91}

@pipeline(dynamic=True)
def gated_pipeline() -> None:
    metrics = evaluate()
    if metrics.load()["accuracy"] >= 0.9:
        promote_model(metrics=metrics)
```

**Migration warning:** this is a real semantic rewrite. Do not keep writing JSON files just to emulate `PropertyFile` unless you are preserving an external contract.

## 4. `ConditionStep` -> dynamic pipeline control flow

### SageMaker SDK

```python
from sagemaker.workflow.condition_step import ConditionStep
from sagemaker.workflow.conditions import ConditionGreaterThan
from sagemaker.workflow.functions import JsonGet

gate = ConditionStep(
    name="AccuracyGate",
    conditions=[
        ConditionGreaterThan(
            left=JsonGet(step_name=step_eval.name, property_file=evaluation_report, json_path="metrics.accuracy"),
            right=0.9,
        )
    ],
    if_steps=[step_register],
    else_steps=[step_fail],
)
```

### ZenML

```python
from zenml import pipeline, step

@step
def evaluate_model() -> dict[str, float]:
    return {"accuracy": 0.93}

@step
def fail_pipeline(message: str) -> None:
    raise RuntimeError(message)

@pipeline(dynamic=True)
def training_pipeline() -> None:
    metrics = evaluate_model()
    if metrics.load()["accuracy"] >= 0.9:
        register_model(metrics=metrics)
    else:
        fail_pipeline(message="Accuracy below deployment threshold")
```

**Migration note:** SageMaker evaluates conditions on backend placeholders. ZenML evaluates regular Python logic. Use `.load()` only for small control artifacts.

## 5. S3 channels and runtime settings -> `SagemakerOrchestratorSettings`

### SageMaker SDK

```python
processor.run(
    code="preprocess.py",
    inputs=[ProcessingInput(source="s3://bucket/raw", destination="/opt/ml/processing/input")],
    outputs=[ProcessingOutput(source="/opt/ml/processing/output", destination="s3://bucket/output")],
)
```

### ZenML

```python
from zenml import step
from zenml.integrations.aws.flavors.sagemaker_orchestrator_flavor import (
    SagemakerOrchestratorSettings,
)

channel_settings = SagemakerOrchestratorSettings(
    instance_type="ml.m5.large",
    input_data_s3_mode="File",
    input_data_s3_uri={"raw": "s3://bucket/raw"},
    output_data_s3_mode="EndOfJob",
    output_data_s3_uri="s3://bucket/output",
    keep_alive_period_in_seconds=300,
)

@step(settings={"orchestrator": channel_settings})
def preprocess_from_channels() -> None:
    ...
```

**Migration warning:** preserving SageMaker channels is useful when exact SageMaker job semantics matter, but it is less portable than artifact-first ZenML.

## 6. `TuningStep` -> explicit HPO step

### SageMaker SDK

```python
from sagemaker.tuner import HyperparameterTuner
from sagemaker.workflow.steps import TuningStep

tuner = HyperparameterTuner(estimator=estimator, objective_metric_name="validation:rmse", ...)
step_tune = TuningStep(name="TuneModel", step_args=tuner.fit(inputs={"train": train_uri}))
```

### ZenML

```python
from zenml import step

@step
def tune_model(train_uri: str) -> dict[str, str]:
    # Migration note: this keeps SageMaker HPO explicitly instead of pretending
    # the original TuningStep is equivalent to a normal training step.
    return {"best_model_uri": "s3://bucket/best-model.tar.gz"}
```

**Migration warning:** never silently translate `TuningStep` into a single `@step` that trains once.

## 7. `TransformStep` -> explicit Batch Transform step

### SageMaker SDK

```python
from sagemaker.transformer import Transformer
from sagemaker.workflow.steps import TransformStep

transformer = Transformer(model_name="my-model", instance_type="ml.m5.large", instance_count=1)
step_transform = TransformStep(
    name="BatchTransform",
    transformer=transformer,
    inputs=TransformInput(data="s3://bucket/batch-input"),
)
```

### ZenML

```python
from zenml import step

@step
def run_batch_transform(model_name: str, input_s3_uri: str) -> str:
    # Migration note: ZenML has no first-class batch transform step.
    # Keep the AWS-native behavior explicit if you need it.
    return "s3://bucket/batch-output"
```

## 8. `ModelStep` / `RegisterModel` -> explicit registry choice

### SageMaker SDK

```python
step_model_registration = ModelStep(
    name="RegisterModel",
    step_args=model.register(
        content_types=["*"],
        response_types=["application/json"],
        inference_instances=["ml.m5.xlarge"],
        transform_instances=["ml.m5.xlarge"],
    ),
)
```

### ZenML

```python
from zenml import step

@step
def register_model_explicitly(model_data_uri: str, model_package_group: str) -> str:
    # Migration note: choose intentionally between:
    # 1. explicit SageMaker registration, or
    # 2. redesigning model governance around ZenML MCP.
    return "arn:aws:sagemaker:eu-west-1:123456789012:model-package/my-model"
```

**Migration warning:** "SageMaker Model Registry == ZenML MCP" is false. They overlap in intent, not semantics.

## 9. Scheduling -> `Schedule(...)` with SageMaker caveats

### SageMaker SDK

```python
from sagemaker.workflow.triggers import PipelineSchedule

pipeline.put_triggers(
    triggers=[PipelineSchedule(cron="0 3 * * ? *", name="daily-training")],
    role_arn="<SCHEDULER_ROLE_ARN>",
)
```

### ZenML

```python
from zenml import pipeline
from zenml.config.schedule import Schedule

@pipeline
def scheduled_pipeline() -> None:
    ...

scheduled_pipeline.with_options(
    schedule=Schedule(cron_expression="0 3 * * *")
)()
```

**Migration warning:** the SageMaker orchestrator supports cron / interval / one-time scheduling, but dynamic pipelines cannot currently be scheduled there, and ZenML does not natively manage schedule updates/deletes on SageMaker.

## 10. `CallbackStep` / `LambdaStep` -> explicit redesign

### SageMaker SDK

```python
step_callback = CallbackStep(...)
step_lambda = LambdaStep(...)
```

### ZenML

```python
from zenml import step

@step
def invoke_lambda_explicitly(payload: dict[str, str]) -> dict[str, str]:
    # TODO(migration): preserve AWS-native Lambda semantics explicitly.
    return {"status": "submitted"}
```

**Migration warning:** do not pretend these are normal ZenML primitives. Keep the AWS integration explicit or redesign the orchestration boundary.
