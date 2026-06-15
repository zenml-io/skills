# Kedro -> ZenML Concept Map

Complete mapping of Kedro concepts to their closest ZenML equivalents. Each entry is classified as **direct** (clean 1:1 mapping), **approximate** (similar intent, different semantics), or **absent** (no real ZenML equivalent -- requires redesign).

## Table of Contents

- [Core Primitives](#core-primitives)
- [Data Catalog and Datasets](#data-catalog-and-datasets)
- [Configuration](#configuration)
- [Execution](#execution)
- [Hooks](#hooks)
- [Plugins and Deployment](#plugins-and-deployment)
- [Orchestrator Scheduling Support](#orchestrator-scheduling-support)

## Core Primitives

| Kedro Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `node()` / `Node` | `@step` | approximate | Both wrap Python functions as DAG units, but Kedro binds dataset names while ZenML binds typed artifacts and parameters. |
| Plain node function | Plain Python function under `@step` | approximate | Business logic often migrates cleanly, but type hints and artifact semantics become part of the runtime contract. |
| String dataset names in `inputs` / `outputs` | Python variables connecting step calls | absent | ZenML has no catalog-name indirection for normal internal edges. |
| Multiple node inputs / outputs | Step args plus tuple returns with `Annotated[...]` names | direct | Capability exists cleanly in both systems. |
| Node `tags` | Run, artifact, and resource tags | approximate | ZenML has tags, but not Kedro's tag-based step-selection semantics. |
| `Pipeline` | `@pipeline` | approximate | Both define DAGs, but Kedro composes through named datasets while ZenML composes through artifact dependencies. |
| `pipeline()` helper | Python composition inside a `@pipeline` | approximate | Reuse intent is similar; the mechanics are explicit rather than namespace-driven. |
| `Pipeline(..., inputs=..., outputs=..., parameters=...)` remapping | Wrapper pipeline or helper function with explicit args | absent | No built-in ZenML DSL for dataset-name remapping. |
| `namespace=` | No direct equivalent | absent | ZenML reuse is explicit composition plus invocation IDs, not automatic dataset renaming. |
| `prefix_datasets_with_namespace` | No direct equivalent | absent | There is no ZenML switch that prefixes selected edges automatically. |
| `pipeline_registry.py` | Normal Python module organization | approximate | Kedro imposes a registry contract; ZenML does not. |
| `__default__` pipeline | No direct equivalent | absent | In ZenML, you explicitly call the pipeline you want to run. |
| Opinionated Kedro project structure | Flexible repo layout + stack config | absent | ZenML is much less prescriptive about directory structure. |
| `kedro pipeline create` | Internal scaffolding or template workflow | approximate | Similar intent, no core ZenML equivalent. |
| `kedro new --starter` | Internal repo template or Cookiecutter | approximate | Similar project bootstrap goal, different toolchain. |

## Data Catalog and Datasets

| Kedro Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `DataCatalog` | No core equivalent | absent | ZenML has an artifact store plus materializers, not a registry of named datasets with IO config. |
| `catalog.yml` | No single equivalent | absent | Its contents fan out across loader/exporter steps, YAML config, materializers, secrets, and stack settings. |
| `pandas.CSVDataset` | `pd.DataFrame` artifact plus explicit CSV loader/exporter step | approximate | External CSV IO becomes an explicit boundary. |
| `pandas.ParquetDataset` | DataFrame artifact plus explicit parquet boundary step | approximate | Artifact storage is not the same contract as "write parquet to this path". |
| `json.JSONDataset` | `dict` / `list` artifact plus explicit JSON export if needed | approximate | Good fit for configuration payloads or reports. |
| `pickle.PickleDataset` | Typed artifact with built-in or custom materializer | approximate | Do not assume "pickle file on disk" and "artifact in artifact store" are equivalent. |
| `pandas.ExcelDataset` | DataFrame artifact plus explicit Excel export step | approximate | Best treated as a reporting boundary. |
| `spark.SparkDataset` | Spark-based step code, often with step-operator support | approximate | Similar compute intent, not a catalog alias. |
| `pillow.ImageDataset` | Image artifact type or explicit image-file IO | approximate | Similar use case, different storage contract. |
| `PartitionedDataset` | Partition-discovery step plus dynamic pipeline or explicit loop | approximate | No built-in catalog wrapper that transparently exposes partitions. |
| `IncrementalDataset` | Checkpoint artifact plus explicit "what is new?" logic | approximate | Checkpoint semantics need redesign. |
| `MemoryDataset` | Normal upstream step output | approximate | Closest match, but ZenML persists artifacts by default. |
| `SharedMemoryDataset` | No equivalent | absent | ZenML does not expose a shared-memory dataset edge. |
| `SharedMemoryDataCatalog` | No equivalent | absent | Shared memory is not ZenML's artifact-sharing model. |
| Dataset factories and catch-all patterns | No equivalent | absent | ZenML does not resolve dataset names through catalog patterns. |
| Default runtime dataset patterns | No equivalent | absent | Kedro can synthesize runtime datasets; ZenML step outputs are explicit. |
| `versioned: true` | Artifact versioning, explicit external version handling, or both | approximate | Similar goal, different storage and retrieval semantics. |
| `credentials:` on datasets | Secrets, env vars, service connectors, stack auth | approximate | No dataset-local credential injection field in ZenML. |
| `load_args`, `save_args`, `fs_args` | Step code, materializer code, or step settings | approximate | These become implementation details, not catalog metadata. |
| Transcoding such as `dataset@pandas` / `dataset@spark` | No equivalent | absent | Must be redesigned explicitly. |
| Custom dataset via `AbstractDataset` / `AbstractVersionedDataset` | Loader/exporter code or custom materializer | approximate | Depends on whether the class is a storage adapter or a true artifact type. |
| `kedro-datasets` package | No single equivalent | absent | Migrate class by class as "boundary IO adapter" vs "artifact serialization concern". |
| `KedroDataCatalog` experimental feature | No migration target | absent | Normalize back to ordinary catalog behavior before migrating. |
| `kedro catalog describe-datasets` | No direct equivalent | absent | ZenML has artifact and stack inspection, but not catalog inspection because there is no `DataCatalog`. |
| `kedro catalog list-patterns` | No direct equivalent | absent | Pattern-driven dataset resolution has no core ZenML equivalent. |
| `kedro catalog resolve-patterns` | No direct equivalent | absent | Expand naming patterns into explicit boundaries before migrating. |

## Configuration

| Kedro Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `conf/base/` | Shared YAML config files | approximate | Similar intent, different loader contract. |
| `conf/local/` | Secrets, env-specific config, local stack settings | approximate | Similar separation of shared vs local concerns, different mechanics. |
| `OmegaConfigLoader` | No core equivalent | absent | ZenML YAML config does not replicate Kedro's OmegaConf loader behavior. |
| `globals.yml` | No core equivalent | absent | Use normal YAML, env vars, or your own config layer. |
| `parameters.yml` | Pipeline or step parameters in YAML config | approximate | Friendly migration, but wiring becomes explicit in function signatures. |
| `params:` prefix | Typed step or pipeline argument | approximate | Same intent, different syntax and visibility. |
| Runtime override via `--params` or resolvers | Runtime pipeline args or YAML override | approximate | ZenML has runtime override precedence, but not Kedro's resolver model. |
| `credentials.yml` | ZenML secrets, env vars, service connectors | approximate | Same responsibility, different mechanism. |
| Environment-specific overrides | Multiple config files + active stack switching | approximate | Use `dev.yaml`, `prod.yaml`, and stack-specific settings. |
| Custom OmegaConf resolvers | No core equivalent | absent | Re-implement only if the project depends on them heavily. |
| `--config` / `--conf-source` | `.with_options(config_path=\"...\")` | approximate | Similar operational use, different loader architecture. |

## Execution

| Kedro Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `kedro run` | Call a pipeline function | approximate | Same goal, different runtime model. |
| `kedro run --pipeline=...` | Call a different ZenML pipeline | direct | Conceptually straightforward. |
| `kedro run --node` / `--nodes` | Run a step directly or define a smaller pipeline | approximate | Useful during development, but not a drop-in slicing feature. |
| `kedro run --tags` | No direct equivalent | absent | ZenML tags do not implement Kedro-style execution selection. |
| `kedro run --namespaces` | No direct equivalent | absent | No runtime namespace selection mechanism. |
| `--from-nodes`, `--to-nodes`, `--from-inputs`, `--to-outputs` | No direct equivalent | absent | Use smaller pipelines, dedicated backfill entrypoints, or artifact reuse. |
| `--only-missing-outputs` | No direct equivalent | absent | Closest tools are caching and artifact reuse, but semantics differ. |
| `SequentialRunner` | Local sequential orchestration | approximate | Closest operational match. |
| `ParallelRunner` | Orchestrator-managed parallel branches | approximate | Isolation model is very different. |
| `ThreadRunner` | No named equivalent | approximate | There is no ZenML "thread runner" abstraction. |
| Async dataset load/save | Async pipeline execution | approximate | Similar wording, different semantics. |
| Runner choice by CLI | Active stack and orchestrator choice | approximate | In ZenML the execution environment is a stack concern. |

## Hooks

| Kedro Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `after_context_created` | No direct equivalent | absent | No matching lifecycle hook in ZenML core. |
| `after_catalog_created` | No direct equivalent | absent | There is no `DataCatalog` lifecycle in ZenML. |
| `before_pipeline_run` | Wrapper code before pipeline invocation | approximate | Not a formal ZenML hook. |
| `after_pipeline_run` | `@pipeline(on_success=...)` | approximate | Similar high-level use, less context-rich. |
| `on_pipeline_error` | `@pipeline(on_failure=...)` | approximate | Closest formal equivalent. |
| `before_node_run` | No direct equivalent | absent | No official before-step hook in ZenML. |
| `after_node_run` | `@step(on_success=...)` | approximate | Similar outcome hook, different payload. |
| `on_node_error` | `@step(on_failure=...)` | approximate | Closest formal equivalent. |
| `before_dataset_loaded` | No direct equivalent | absent | Dataset lifecycle hooks are a Kedro-specific strength. |
| `after_dataset_loaded` | No direct equivalent | absent | Same issue. |
| `before_dataset_saved` | No direct equivalent | absent | Same issue. |
| `after_dataset_saved` | No direct equivalent | absent | Same issue. |
| `before_command_run` | No direct equivalent | absent | ZenML CLI / Python execution does not expose a matching hook. |
| `after_command_run` | No direct equivalent | absent | Same issue. |

**Pipeline-level hooks fire differently on dynamic vs static pipelines.** `@pipeline(on_success=...)`/`@pipeline(on_failure=...)` fire once at the run level only on a dynamic pipeline (`@pipeline(dynamic=True)`). On a static pipeline the same kwargs are per-step defaults that every step inherits, and they never fire once at the run level. To reproduce Kedro's single `after_pipeline_run`/`on_pipeline_error` callback on a static pipeline, attach the hook to one terminal step instead.

## Plugins and Deployment

| Kedro Concept | ZenML Equivalent | Mapping | Notes |
|---|---|:---:|---|
| `kedro-airflow` | Airflow orchestrator stack component | approximate | Kedro exports a Kedro project to Airflow DAGs; ZenML orchestrates steps directly on Airflow. |
| `kedro-kubeflow` | Kubeflow orchestrator stack component | approximate | Same destination platform, different architecture. |
| `kedro-vertexai` | Vertex orchestrator stack component | approximate | Same destination, different execution model. |
| `kedro-docker` | `DockerSettings` plus stack-driven containerization | approximate | Kedro packages the whole project; ZenML usually builds execution images per stack / pipeline context. |
| `kedro-mlflow` | ZenML MLflow experiment tracker | approximate | Similar tracking destination, different integration shape. |
| Kedro-Viz pipeline graph | ZenML dashboard DAG and artifact views | approximate | Similar observability goal, different metadata model. |
| Kedro-Viz native experiment tracking | No direct equivalent | absent | Use experiment trackers and metadata logging instead. |
| `kedro-datasets` plugin ecosystem | No single equivalent | absent | ZenML has integrations and materializers, but not a catalog plugin layer of the same shape. |
| Platform-specific Kedro deployment plugins generally | ZenML stacks, orchestrators, step operators, schedules | approximate | Kedro often exports to another platform; ZenML treats the platform as part of the stack. |

## Orchestrator Scheduling Support

ZenML scheduling is a wrapper around native orchestrator scheduling, not its own scheduler. Support therefore varies by orchestrator.

| ZenML Orchestrator | Scheduling | Types Supported / Notes |
|---|:---:|---|
| Local | No | No native scheduling |
| Local Docker | No | No native scheduling |
| Airflow | Yes | Cron, interval |
| AzureML | Yes | Cron, interval |
| Databricks | Yes | Cron only |
| HyperAI | Yes | Cron, one-time |
| Kubeflow | Yes | Cron, interval |
| Kubernetes | Yes | Cron |
| SageMaker | Yes | Cron, interval, one-time |
| SkyPilot | No | No native scheduling |
| Tekton | No | No native scheduling |
| Vertex AI | Yes | Cron |

## Practical Translation Rule

When in doubt, use this order of reasoning:

1. Start from `catalog.yml`, not the nodes.
2. Decide whether each dataset is a boundary or an internal handoff.
3. Convert internal handoffs into artifacts.
4. Convert boundary reads and writes into explicit steps.
5. Flag anything that depends on hidden catalog behavior, namespaces, slicing, or dataset lifecycle hooks.
