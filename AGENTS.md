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

## Commit & Pull Request Guidelines
Recent commits use short, imperative subjects (for example: `Add zenml-scoping...`, `Fix plugin agent format...`).

- Keep commits scoped to one logical change (single plugin or documentation update).
- Stage files selectively (`git add <path>`), not broad adds.
- PRs should include: what changed, why, affected paths, and validation commands run.
- Link related issues/tasks when available, and call out marketplace entry changes explicitly.
