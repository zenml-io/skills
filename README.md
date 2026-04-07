# ZenML Agent Skills

Modular AI coding agent skills for ZenML workflows. Add the marketplace to your AI coding tool and install skills that guide you through implementing ZenML features.

<img width="991" height="973" alt="Screenshot of the quick wins skill in action inside Claude Code" src="https://github.com/user-attachments/assets/a6e54013-3033-41fd-abfa-e29d7b324e32" />

## Quick Start

### Claude Code

```bash
# Add the ZenML skills marketplace
/plugin marketplace add zenml-io/skills

# Install a skill (e.g. quick-wins)
/plugin install zenml-quick-wins@zenml

# Use it — navigate to your ZenML project and run:
/zenml-quick-wins
```

[Plugin docs](https://code.claude.com/docs/en/plugins) · [Marketplace docs](https://code.claude.com/docs/en/plugin-marketplaces)

### OpenAI Codex CLI

Ask the built-in skill installer to fetch skills from this repo:

```
$skill-installer install the zenml skills from github.com/zenml-io/skills
```

[Codex skills docs](https://developers.openai.com/codex/skills#install-skills)

### Cursor

1. Open **Cursor Settings** (`Cmd+Shift+J` / `Ctrl+Shift+J`)
2. Navigate to **Rules** → **Project Rules** → **Add Rule**
3. Choose **Remote Rule (GitHub)** and enter `https://github.com/zenml-io/skills`

[Cursor skills docs](https://cursor.com/docs/context/skills#installing-skills-from-github)


## Available Skills

| Skill | Description | Install |
|-------|-------------|---------|
| `zenml-quick-wins` | Analyze your setup, recommend high-impact improvements, and implement features like metadata logging, experiment tracking, alerts, and model governance | `/plugin install zenml-quick-wins@zenml` |
| `zenml-scoping` | Scope and decompose ML workflow ideas into realistic ZenML pipeline architectures through a structured interview process | `/plugin install zenml-scoping@zenml` |
| `zenml-pipeline-authoring` | Author ZenML pipelines with steps, artifacts, Docker settings, materializers, metadata, secrets, YAML config, and visualizations | `/plugin install zenml-pipeline-authoring@zenml` |
| `zenml-airflow-migration` | Migrate Apache Airflow DAGs to idiomatic ZenML pipelines with concept mapping, code translation, severity-classified flagging, and redesign suggestions | `/plugin install zenml-airflow-migration@zenml` |
| `zenml-argo-migration` | Migrate Argo Workflows to idiomatic ZenML pipelines with concept mapping, code translation, Kubernetes-native pattern flagging, and redesign guidance | `/plugin install zenml-argo-migration@zenml` |
| `zenml-databricks-migration` | Migrate Databricks Workflows (Lakeflow Jobs) to idiomatic ZenML pipelines with notebook refactoring, concept mapping, code translation, and unsupported pattern flagging | `/plugin install zenml-databricks-migration@zenml` |
| `zenml-prefect-migration` | Migrate Prefect flows and workflows to idiomatic ZenML pipelines with concept mapping, dynamic-execution analysis, code translation, and unsupported pattern flagging | `/plugin install zenml-prefect-migration@zenml` |
| `zenml-vertexai-migration` | Migrate Vertex AI Pipelines (KFP v2 / PipelineJob workflows) to idiomatic ZenML pipelines with concept mapping, artifact-contract translation, GCPC rewrite guidance, and unsupported pattern flagging | `/plugin install zenml-vertexai-migration@zenml` |
| `zenml-azureml-migration` | Migrate Azure Machine Learning SDK v2 pipelines to idiomatic ZenML pipelines with concept mapping, Azure-aware compute translation, code patterns, and unsupported pattern flagging | `/plugin install zenml-azureml-migration@zenml` |
| `zenml-dagster-migration` | Migrate Dagster assets, ops, jobs, and asset-graph workflows to idiomatic ZenML pipelines with concept mapping, pipeline-boundary planning, code translation, and unsupported pattern flagging | `/plugin install zenml-dagster-migration@zenml` |
| `zenml-flyte-migration` | Migrate Flyte workflows, tasks, and LaunchPlans to idiomatic ZenML pipelines with concept mapping, special-type planning, code translation, and unsupported pattern flagging | `/plugin install zenml-flyte-migration@zenml` |
| `zenml-kedro-migration` | Migrate Kedro pipelines and projects to idiomatic ZenML pipelines with Data Catalog to artifact translation, concept mapping, code translation, and unsupported pattern flagging | `/plugin install zenml-kedro-migration@zenml` |
| `zenml-metaflow-migration` | Migrate Metaflow flows to idiomatic ZenML pipelines with FlowSpec mapping, artifact translation, control-flow redesign notes, and unsupported pattern flagging | `/plugin install zenml-metaflow-migration@zenml` |
| `zenml-sagemaker-migration` | Migrate Amazon SageMaker Pipelines to idiomatic ZenML pipelines with concept mapping, artifact-model rewrites, runtime-setting translation, and unsupported pattern flagging | `/plugin install zenml-sagemaker-migration@zenml` |

## Coming Soon

- **Debugging** — Investigate pipeline failures and performance issues

## Combine with MCP Servers

For the best AI-assisted ZenML experience, pair skills with MCP servers:

```bash
# Add ZenML docs MCP (Claude Code)
claude mcp add zenmldocs --transport http https://docs.zenml.io/~gitbook/mcp
```

This gives your AI assistant both **structured workflows** (skills) and **doc-grounded answers** (MCP).

## Learn More

- [ZenML Documentation](https://docs.zenml.io)
- [LLM Tooling Reference](https://docs.zenml.io/reference/llms-txt) — Full guide to MCP servers, llms.txt, and skills
