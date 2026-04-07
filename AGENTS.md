# Repository Guidelines

## Project Structure & Module Organization
This repository is a skills marketplace, not an application codebase.

- `.claude-plugin/marketplace.json`: top-level marketplace manifest listing all plugins.
- `skills/<plugin>/plugin.json`: plugin metadata and the relative skills path (`"./skills/"`).
- `skills/<plugin>/skills/<skill-name>/SKILL.md`: primary skill instructions (Markdown with YAML frontmatter).
- Supporting docs may live beside each skill (for example `references/*.md` or `agents/*.md`).
- `design/` is for local design artifacts and should not be committed.

## Build, Test, and Development Commands
There is no build pipeline or runtime service in this repo. Use validation and inspection commands instead:

- `jq . .claude-plugin/marketplace.json`: validate marketplace JSON syntax.
- `jq . skills/<plugin>/plugin.json`: validate a plugin manifest.
- `rg --files skills`: list skill files and confirm expected paths.
- `git status`: verify only intended files are staged/modified.

## Coding Style & Naming Conventions
- Use clear, instructional Markdown in `SKILL.md` files with frontmatter.
- Set `name` to a kebab-case identifier.
- Write `description` as concise purpose plus “Use when …” guidance.
- Keep plugin and skill folder names kebab-case (for example `zenml-pipeline-authoring`).
- Follow existing JSON style: 2-space indentation, double-quoted keys/strings.
- Prefer concrete, actionable steps over abstract guidance.
- Keep ZenML doc links current (`https://docs.zenml.io/...`).

## Testing Guidelines
No automated test framework is configured here. Treat structure validation as required:

1. Confirm skill file location: `skills/<plugin>/skills/<skill>/SKILL.md`.
2. Confirm `plugin.json` includes `"skills": "./skills/"`.
3. Confirm plugin registration in `.claude-plugin/marketplace.json`.
4. Manually preview changed Markdown for heading and code fence rendering.

## Plugin-Specific Notes

### zenml-airflow-migration
- The skill uses three reference files (`references/concept-map.md`, `references/code-patterns.md`, `references/gaps-and-flags.md`) for progressive disclosure — SKILL.md stays focused on the workflow, references are loaded on demand.
- Post-migration, the skill chains to `zenml-quick-wins` and optionally `zenml-pipeline-authoring` for further enhancement.
- The migration report template includes a "What You Get for Free" section and community Slack support for complex migrations with many unsupported patterns.
- Post-migration, the skill suggests running `/simplify` to clean up migration comments and reduce duplication, and offers to open GitHub issues on `zenml-io/zenml` for genuine feature gaps.

### zenml-databricks-migration
- Same three-reference-file architecture as the airflow-migration skill (`references/concept-map.md`, `references/code-patterns.md`, `references/gaps-and-flags.md`).
- Handles Databricks-specific challenges: notebook refactoring (classifying notebooks by risk level based on magics, dbutils usage, Spark state), heterogeneous task types (notebook, wheel, SQL, dbt, condition, for_each, run_job), and platform-coupled features (Unity Catalog, DBFS, managed identity).
- The `gaps-and-flags.md` reference includes a notebook classification guide (auto-refactorable / semi-automatic / manual refactor) that the main SKILL.md references during Phase 1 analysis.
- Post-migration, chains to `zenml-quick-wins`, suggests `/simplify`, and offers GitHub issue creation for genuine feature gaps.
- Migration report includes a "Notebook Refactoring Summary" section unique to this skill.

### zenml-argo-migration
- Uses the same three-reference-file progressive-disclosure architecture as the other migration skills (`references/concept-map.md`, `references/code-patterns.md`, `references/gaps-and-flags.md`).
- The core Argo migration challenge is Kubernetes-native workflow semantics: YAML CRDs, pod topology, status-based dependency logic, and volume/sidecar assumptions do not map 1:1 to ZenML's Python artifact-DAG model.
- Treat enhanced `depends`, `containerSet`, shared PVC / `emptyDir` contracts, sidecars, synchronization locks, and Argo Events graphs as redesign-first patterns rather than safe auto-translations.
- Post-migration, chains to `zenml-quick-wins`, suggests `/simplify`, and offers GitHub issue creation or community escalation for genuine feature gaps.

### zenml-prefect-migration
- Same three-reference-file architecture as the airflow-migration and databricks-migration skills (`references/concept-map.md`, `references/code-patterns.md`, `references/gaps-and-flags.md`).
- The core Prefect migration challenge is semantic, not syntactic: dynamic execution, state inspection, and control-plane features do not map 1:1 to ZenML's artifact-DAG model.
- Always classify runtime branching, runtime fan-out, `return_state=True`, `allow_failure()`, pause/suspend, task-runner semantics, global concurrency/rate limiting, and transaction patterns explicitly.
- Treat Blocks as a decomposition problem (secrets + service connectors + stack config + YAML/params), not a single-object mapping.
- Post-migration, chains to `zenml-quick-wins`, suggests `/simplify`, and offers GitHub issue creation for genuine feature gaps.

### zenml-azureml-migration
- Same three-reference-file architecture as the other migration skills (`references/concept-map.md`, `references/code-patterns.md`, `references/gaps-and-flags.md`) with SKILL.md staying workflow-focused.
- The skill is explicitly scoped to **Azure Machine Learning SDK v2**. If source material is still on SDK v1, normalize to v2 concepts before attempting a ZenML migration.
- The skill treats the "keep AzureML" path as first-class: ZenML authors the workflow while AzureML can remain the execution plane through the AzureML orchestrator.
- Preserve explicit caveats around unsafe control-flow auto-translation (`if_else`, `do_while`, `parallel_for`, `set_pipeline_controller_configurations`) and do not present them as safe 1:1 migrations.
- Post-migration, chains to `zenml-quick-wins`, suggests `/simplify`, optionally routes deeper customization to `zenml-pipeline-authoring`, and offers GitHub issue creation for genuine feature gaps.

### zenml-dagster-migration
- Same three-reference-file architecture as the airflow-migration and databricks-migration skills (`references/concept-map.md`, `references/code-patterns.md`, `references/gaps-and-flags.md`).
- The core Dagster migration challenge is semantic, not syntactic: asset graphs and materialization boundaries do not map 1:1 to ZenML pipeline execution boundaries.
- The skill requires an explicit pipeline-boundary decision before code generation: single pipeline, multiple pipelines, or partial migration plus redesign report.
- The `gaps-and-flags.md` reference is especially important for IO managers with business logic, partitions/backfills, sensors, declarative automation, freshness policies, and asset-selection semantics.
- Post-migration, chains to `zenml-quick-wins`, suggests `/simplify`, and offers GitHub issue creation for genuine feature gaps.
- Migration report includes a "Pipeline Boundary Decisions" section unique to this skill.

### zenml-flyte-migration
- Uses the same three-reference-file progressive-disclosure architecture as the other migration skills (`references/concept-map.md`, `references/code-patterns.md`, `references/gaps-and-flags.md`).
- The core Flyte migration challenge is preserving typed workflow contracts and registered-execution semantics while translating `@dynamic`, `map_task()`, `conditional()`, `LaunchPlan`, and special transport types (`FlyteFile`, `StructuredDataset`, `FlyteSchema`).
- Treat `@eager`, `ContainerTask`, reference entities, checkpointing, interruptible semantics, and Union-specific features as redesign-first or high-caveat patterns.
- Post-migration, chains to `zenml-quick-wins`, suggests `/simplify`, and offers GitHub issue creation for genuine feature gaps.

### zenml-kedro-migration
- Uses the same three-reference-file progressive-disclosure architecture as the other migration skills (`references/concept-map.md`, `references/code-patterns.md`, `references/gaps-and-flags.md`).
- The hard part of Kedro migration is not node code but the Data Catalog contract: loader/exporter boundaries, dataset configuration, versioning, credentials, and operational semantics must become explicit in ZenML.
- Preserve explicit caveats around transcoding, dataset lifecycle hooks, namespace/remapping behavior, slicing semantics, and shared-memory runner assumptions.
- Post-migration, chains to `zenml-quick-wins`, suggests `/simplify`, and offers GitHub issue creation for genuine feature gaps.

### zenml-metaflow-migration
- Uses the same three-reference-file progressive-disclosure architecture as the other migration skills (`references/concept-map.md`, `references/code-patterns.md`, `references/gaps-and-flags.md`).
- Metaflow is superficially close to ZenML, so the main danger is silent approximation: `self.*` artifact propagation, join semantics, `foreach`, `merge_artifacts`, `@catch`, resume/checkpoint behavior, and Outerbounds features must be classified explicitly.
- Dynamic-pipeline translations should stay honest about orchestrator support and the `.load()` vs `.chunk(idx)` distinction when manual dynamic loops are involved.
- Post-migration, chains to `zenml-quick-wins`, suggests `/simplify`, and offers GitHub issue creation or community escalation for genuine feature gaps.

### zenml-sagemaker-migration
- Uses the same three-reference-file progressive-disclosure architecture as the other migration skills (`references/concept-map.md`, `references/code-patterns.md`, `references/gaps-and-flags.md`).
- Centers the "keep SageMaker, change the authoring model" migration path: ZenML `@pipeline` / `@step` authoring with the SageMaker orchestrator and `SagemakerOrchestratorSettings` when the user wants to preserve AWS-native runtime behavior.
- Handles SageMaker-specific challenges: step-type DSL migration (`ProcessingStep`, `TrainingStep`, `ConditionStep`, `TuningStep`, `TransformStep`, `ModelStep`), placeholder/data-model rewrites (`PropertyFile`, `JsonGet`, step `.properties`), and AWS-service-coupled steps (`CallbackStep`, `LambdaStep`, Clarify, Model Monitor, EMR, Notebook Jobs, Autopilot).
- The `gaps-and-flags.md` reference is the safety layer for high-risk semantic mismatches such as dynamic-pipeline scheduling on the SageMaker orchestrator, Model Registry vs ZenML MCP, Feature Store parity claims, and warm-pool / output-mode trade-offs.
- Post-migration, chains to `zenml-quick-wins`, suggests `/simplify`, and offers GitHub issue creation or community escalation for genuine feature gaps.
- Migration report includes a "SageMaker Runtime Mapping" section to make the keep-SageMaker runtime choices explicit.

### zenml-vertexai-migration
- Uses the same three-reference-file progressive-disclosure architecture as the other migration skills (`references/concept-map.md`, `references/code-patterns.md`, `references/gaps-and-flags.md`).
- Centers the "keep Vertex as execution plane, adopt ZenML as control plane" migration path: prefer ZenML-native rewrites on the Vertex orchestrator, with black-box `PipelineJob` wrapping only as an explicitly documented fallback.
- Treat artifact path/URI contracts, GCPC operators, compiled-template workflows, `dsl.ExitHandler`, and schedule-lifecycle parity as high-caveat or redesign-first areas.
- Post-migration, chains to `zenml-quick-wins`, suggests `/simplify`, and offers GitHub issue creation for genuine feature gaps.

## Commit & Pull Request Guidelines
Recent commits use short, imperative subjects (for example: `Add zenml-scoping...`, `Fix plugin agent format...`).

- Keep commits scoped to one logical change (single plugin or documentation update).
- Stage files selectively (`git add <path>`), not broad adds.
- PRs should include: what changed, why, affected paths, and validation commands run.
- Link related issues/tasks when available, and call out marketplace entry changes explicitly.
