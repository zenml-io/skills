# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is the **ZenML Skills Marketplace** — a collection of modular AI coding agent skills for ZenML workflows. Skills are plugins that guide developers through implementing ZenML features like metadata logging, experiment tracking, pipeline architecture scoping, and pipeline authoring.

## Current Plugins

| Plugin | Description |
|--------|-------------|
| `zenml-quick-wins` | Guided implementation of ZenML MLOps best practices (metadata, experiment tracking, alerters, scheduling, secrets, Model Control Plane) |
| `zenml-scoping` | Scope and decompose ML workflow ideas into realistic ZenML pipeline architectures via an interview process |
| `zenml-pipeline-authoring` | Author ZenML pipelines with steps, artifacts, Docker settings, materializers, YAML config, and more |

## Architecture

```
.claude-plugin/
  marketplace.json          # Marketplace manifest listing all plugins
skills/
  zenml-quick-wins/         # Plugin directory
    plugin.json             # Plugin manifest (name, version, skills path)
    agents/                 # Custom subagents (auto-discovered by Claude Code)
      zenml-codebase-analyzer.md
      zenml-stack-investigator.md
    skills/
      quick-wins/
        SKILL.md            # Skill definition and instructions
        quick-wins-catalog.md
  zenml-scoping/
    plugin.json
    skills/
      scoping/
        SKILL.md
  zenml-pipeline-authoring/
    plugin.json
    skills/
      pipeline-authoring/
        SKILL.md
        references/         # Supplementary docs referenced by the skill
          docker-settings.md
          dynamic-pipelines.md
          external-data.md
          materializers.md
          post-creation.md
          yaml-config.md
```

### Key Files

- **`.claude-plugin/marketplace.json`** — Marketplace catalog. Users add this marketplace via `/plugin marketplace add zenml-io/skills`, then install individual plugins with `/plugin install <plugin-name>@zenml`
- **`skills/<plugin>/plugin.json`** — Plugin manifest with name, version, and skills path
- **`skills/<plugin>/skills/<skill>/SKILL.md`** — Skill instructions loaded when the skill is invoked
- **`skills/<plugin>/agents/*.md`** — Custom subagent definitions (auto-discovered at plugin root level)
- **`skills/<plugin>/skills/<skill>/references/*.md`** — Supplementary documentation that a skill's SKILL.md can reference

### Adding a New Skill

1. Create a new plugin directory: `skills/<plugin-name>/`
2. Add `plugin.json` with `"skills": "./skills/"` (and name, version, description, etc.)
3. Create skill directory: `skills/<plugin-name>/skills/<skill-name>/`
4. Write `SKILL.md` with frontmatter (`name`, `description`) and instructions
5. Optionally add `references/` subdirectory for supplementary docs
6. Optionally add `agents/` at the plugin root for custom subagents
7. Register the plugin in `.claude-plugin/marketplace.json` under the `plugins` array

### Skill File Format

Skills use markdown with YAML frontmatter:

```markdown
---
name: skill-name
description: >-
  Multi-line description explaining when to use the skill.
  Use when: user asks about X, mentions Y, or needs Z.
---

# Skill Title

Workflow instructions, code examples, and guidance.
```

### Agent File Format

Agents use markdown with YAML frontmatter at the plugin root level (`agents/` directory):

```markdown
---
description: >-
  What this agent specializes in and when Claude should invoke it.
capabilities:
  - Capability one
  - Capability two
---

# Agent Title

Detailed system prompt describing the agent's role, search strategy, and output format.
```

## No Build System

This repository has no build, test, or lint commands. It contains only markdown documentation and JSON manifests.

## External Dependencies

Skills reference ZenML documentation at `https://docs.zenml.io/` — keep doc links current when editing skills.

## External Documentation

Anthropic has great documentation and if you're doing something that relates to sharing plugins or the marketplace, do read their guides:

- General high-level guide: https://code.claude.com/docs/en/plugins
- Detailed plugin reference: https://code.claude.com/docs/en/plugins-reference
- Full skills documentation: https://code.claude.com/docs/en/skills
- Custom subagents docs: https://code.claude.com/docs/en/sub-agents
- Plugin marketplaces: https://code.claude.com/docs/en/plugin-marketplaces
