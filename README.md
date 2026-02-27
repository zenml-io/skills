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

## Coming Soon

- **Debugging** — Investigate pipeline failures and performance issues
- **Migration** — Migrate from other MLOps platforms to ZenML

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
