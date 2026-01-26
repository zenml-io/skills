# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is the **ZenML Skills Marketplace** — a collection of modular AI coding agent skills for ZenML workflows. Skills are plugins that guide developers through implementing ZenML features like metadata logging, experiment tracking, alerts, and model governance.

## Architecture

```
.claude-plugin/
  marketplace.json    # Marketplace manifest listing all plugins
skills/
  zenml-quick-wins/   # Each plugin is a subdirectory
    plugin.json       # Plugin manifest
    skills/           # Skills directory
      quick-wins/     # Each skill is a subdirectory
        SKILL.md      # Skill definition and instructions
        *.md          # Supporting documentation
```

### Key Files

- **`.claude-plugin/marketplace.json`** — Marketplace definition, read by `zenml-io/skills` when users run `/plugin marketplace add zenml-io/skills`
- **`skills/<plugin>/plugin.json`** — Plugin manifest with name, version, and skills path
- **`skills/<plugin>/skills/<skill>/SKILL.md`** — Skill instructions loaded when the skill is invoked

### Adding a New Skill

1. Create a new plugin directory: `skills/<plugin-name>/`
2. Add `plugin.json` with `"skills": "./skills/"`
3. Create skill directory: `skills/<plugin-name>/skills/<skill-name>/`
4. Write `SKILL.md` with frontmatter (`name`, `description`) and instructions
5. Register in `.claude-plugin/marketplace.json` under `plugins` array

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

## No Build System

This repository has no build, test, or lint commands. It contains only markdown documentation and JSON manifests.

## External Dependencies

Skills reference ZenML documentation at `https://docs.zenml.io/` — keep doc links current when editing skills.

## External Documentation

Anthropic has great documentation and if you're doing something that relates to sharing plugins or the marketplace, do read their guides:

- general high-level guide: https://code.claude.com/docs/en/plugins
- Detailed plugin reference: https://code.claude.com/docs/en/plugins-reference
- Full skills documentation: https://code.claude.com/docs/en/skills
- Custom subagents docs: https://code.claude.com/docs/en/sub-agents
