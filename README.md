# ZenML Agent Skills

Modular AI coding agent skills for ZenML workflows. Add the marketplace to your AI coding tool and install skills that guide you through implementing ZenML features.

<img width="991" height="973" alt="Screenshot of the quick wins skill in action inside Claude Code" src="https://github.com/user-attachments/assets/a6e54013-3033-41fd-abfa-e29d7b324e32" />

## Quick Start (Claude Code)

```bash
# Step 1: Add the ZenML skills marketplace
/plugin marketplace add zenml-io/skills

# Step 2: Install the quick-wins skill
/plugin install zenml-quick-wins@zenml

# Step 3: Use it! Navigate to your ZenML project and run:
/zenml-quick-wins
```

That's it! The skill will analyze your ZenML setup and guide you through implementing high-impact improvements.

## Supported Tools

| Tool | Add Marketplace |
|------|-----------------|
| [Claude Code](https://code.claude.com/) | `/plugin marketplace add zenml-io/skills` |
| [OpenAI Codex CLI](https://github.com/openai/codex) | `codex plugin add zenml-io/skills` |
| [Cursor](https://cursor.sh) | Via settings configuration |
| [GitHub Copilot](https://github.com/features/copilot) | Agent Skills integration |
| [Gemini CLI](https://github.com/google/gemini-cli) | Skills support |

## Available Skills

| Skill | Description | Install |
|-------|-------------|---------|
| `zenml-quick-wins` | Analyze your setup, recommend high-impact improvements, and implement features like metadata logging, experiment tracking, alerts, and model governance | `/plugin install zenml-quick-wins@zenml` |

## Coming Soon

- **Pipeline creation** — Scaffold new pipelines from templates
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
