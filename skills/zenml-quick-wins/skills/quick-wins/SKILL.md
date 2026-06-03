---
name: zenml-quick-wins
description: >-
  Implements ZenML quick wins to enhance MLOps workflows. Investigates codebase 
  and stack configuration, recommends high-priority improvements, and implements 
  metadata logging, experiment tracking, alerts, scheduling, secrets management,
  tags, git hooks, HTML reports, Model Control Plane setup, live streaming events,
  and resource-pool-aware step requests.
  Use when: user wants to improve their ZenML setup, asks about MLOps best practices,
  mentions "quick wins", wants to enhance pipelines, or needs help with ZenML features
  like experiment tracking, alerting, scheduling, or model governance.
---

# ZenML Quick Wins Implementation

Guides users through discovering and implementing high-impact ZenML features that take ~5 minutes each. Investigates current setup, recommends priorities, and implements chosen improvements.

## Workflow Overview

```
┌─────────────────────┐
│  1. INVESTIGATE     │  Understand current stack + codebase
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│  2. CONFIRM         │  ⏸️ Check understanding with user
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│  3. GATHER CONTEXT  │  ⏸️ Get additional context from user
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│  4. RECOMMEND       │  Prioritize quick wins based on findings
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│  5. PREPARE         │  ⏸️ Verify branch setup before changes
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│  6. IMPLEMENT       │  Apply selected quick wins
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│  7. VERIFY          │  Confirm implementation works
└─────────────────────┘

⏸️ = User checkpoint (uses AskUserQuestion tool)
```

---

## Phase 1: Investigation

**Use subagents to gather information efficiently.** This keeps verbose output out of the main conversation while enabling parallel investigation.

### Recommended: Parallel Subagent Investigation

Spawn **both subagents in parallel** using a single Task tool call with multiple invocations:

```
Use the Task tool to launch BOTH agents simultaneously:

1. zenml-quick-wins:zenml-stack-investigator agent:
   - Prompt: "Investigate the ZenML stack configuration for this project.
     Run all the ZenML CLI commands to understand stacks, components,
     and recent pipeline activity. Return a structured summary."

2. zenml-quick-wins:zenml-codebase-analyzer agent:
   - Prompt: "Analyze the Python codebase for ZenML patterns and quick win
     opportunities. Search for pipeline definitions, current feature usage,
     and areas for improvement. Return a structured summary."
```

Both agents run concurrently and return structured summaries. Synthesize their findings before proceeding to Phase 2.

### What the Subagents Investigate

**zenml-quick-wins:zenml-stack-investigator** (Bash-focused, Haiku model):
- Runs `zenml status`, `zenml stack list`, `zenml stack describe`
- Checks for experiment trackers, alerters, secrets, code repos, models
- Fetches recent pipeline runs
- Returns: Stack configuration, component status, pipeline activity summary

**zenml-quick-wins:zenml-codebase-analyzer** (Read-only, Haiku model):
- Searches for `@pipeline`, `@step` decorators
- Checks for `log_metadata`, `tags=`, `Model(`, `HTMLString`, `Schedule`
- Flags hardcoded credentials (security concern)
- Returns: Feature adoption status, quick win opportunities, security concerns

### Pattern Reference for Analysis

| Pattern | Indicates | Quick Win Opportunity |
|---------|-----------|----------------------|
| `@pipeline` without `tags=` | No tagging | #9 Tags |
| `@step` without `log_metadata` | No metadata | #1 Metadata |
| No `HTMLString` imports | No HTML reports | #11 Reports |
| No `Model()` usage | No model governance | #12 Model Control Plane |
| Hardcoded credentials | Security risk | #7 Secrets |
| No `Schedule` imports | Manual runs | #5 Scheduling |
| `publish(` or token/progress callbacks | Live intermediate output need | #16 Live streaming events |
| `gpu_count`, shared clusters, many dynamic steps | Resource contention risk | #17 Resource pools / requests |

### Fallback: Manual Investigation

If subagents are unavailable, run these commands directly.

**First, activate the Python environment:**

```bash
# Check for uv project
if [ -f "pyproject.toml" ] && command -v uv &> /dev/null; then
    # Use "uv run zenml ..." for all commands below
    echo "Using uv - prefix commands with 'uv run'"
fi

# Or activate venv/conda
source .venv/bin/activate 2>/dev/null || source venv/bin/activate 2>/dev/null || echo "No venv found"

# Verify zenml is available
zenml version
```

**Then run the investigation commands:**

```bash
# Core stack info
zenml status
zenml stack list --output=json
zenml stack describe

# Component details
zenml experiment-tracker list 2>/dev/null || echo "No experiment trackers"
zenml alerter list 2>/dev/null || echo "No alerters configured"
zenml secret list 2>/dev/null || echo "No secrets or no access"
zenml code-repository list 2>/dev/null || echo "No code repos connected"
zenml model list 2>/dev/null || echo "No models registered"

# Recent runs (check for metadata usage)
zenml pipeline runs list --size=10 --output=json 2>/dev/null
```

### MCP Server Check

If ZenML MCP server is available, use it for deeper exploration:

```bash
# Check if zenml MCP tools are available
# If yes, use them to query pipelines, runs, artifacts
```

---

## Phase 2: Confirm Understanding ⏸️

**IMPORTANT: Stop and check in with the user before proceeding.**

After investigation, summarize what you've learned and use the `AskUserQuestion` tool to confirm your understanding. This prevents wasted effort from misunderstandings.

### What to Summarize

Present your findings clearly:

1. **Stack Configuration**
   - Which stacks exist (local, staging, production, etc.)
   - What the active stack contains (orchestrator type, artifact store, etc.)
   - Any connected components (experiment trackers, alerters, secrets stores)

2. **Pipeline Execution Patterns**
   - How pipelines appear to be run (manual vs scheduled vs triggered)
   - If schedules exist, note them explicitly
   - Any orchestrator-specific patterns (Airflow DAGs, Kubeflow pipelines, etc.)

3. **Current Feature Usage**
   - Which ZenML features are already in use (metadata, tags, models, etc.)
   - What's missing that could be quick wins

### Confirmation Questions

Use `AskUserQuestion` with questions like:

**Stack Usage:**
- "I found these stacks: [list]. Which is your primary development stack, and which is production?"
- "Is `default` stack used for local development, or do you use a different one?"

**Pipeline Execution:**
- "I see [N] recent runs. Are these primarily manual runs, or is there a schedule I might have missed?"
- "The pipeline appears to run on [orchestrator]. Is this your main execution environment?"

**Feature Gaps:**
- "I noticed [feature] isn't being used yet. Is that intentional, or an area you'd like to improve?"

### Example Confirmation Message

```markdown
## 📋 Here's what I understand about your ZenML setup:

**Stacks:**
- `default` - Local stack (seems to be for development)
- `aws-production` - AWS stack with SageMaker orchestrator

**Active Stack:** `default` (LocalOrchestrator, local artifact store)

**Recent Activity:**
- 47 pipeline runs in the last month
- All appear to be manual runs (no schedules detected)
- Using `training_pipeline` and `inference_pipeline`

**Current Features:**
- ✅ Basic pipeline structure
- ❌ No metadata logging detected
- ❌ No tags on pipelines
- ❌ No Model Control Plane usage

**Questions for you:**
1. Is `default` your local dev stack and `aws-production` for production?
2. Are these pipelines meant to run on a schedule, or is manual execution intentional?
3. Any other stacks or environments I should know about?
```

Wait for user confirmation before proceeding to Phase 3.

---

## Phase 3: Gather Context ⏸️

**IMPORTANT: Gather tacit knowledge that isn't captured in code.**

Before making recommendations, ask about context that affects implementation choices. Use `AskUserQuestion` to learn about:

### Infrastructure Context

- "Are there any infrastructure constraints or gotchas I should know about?"
- "Any cloud resource limits, network restrictions, or compliance requirements?"
- "Is there a shared artifact store, or does each environment have its own?"

### Development Environment

- "How do you typically develop and test pipelines locally before deploying?"
- "Do you use a specific IDE, and is there any tooling I should be aware of?"
- "Any CI/CD pipelines that interact with ZenML?"

### Team Dynamics

- "Is this a solo project or does a team work on these pipelines?"
- "If team: any conventions or patterns the team follows that I should maintain?"
- "Any upcoming changes or migrations planned that might affect recommendations?"

### Operational Patterns

- "How do you currently monitor pipeline health and failures?"
- "When something breaks, how do you typically find out?"
- "Any preferences for where alerts should go (Slack, email, etc.)?"

### Example Context Gathering

```markdown
## 🔍 Before I make recommendations, some questions:

**Infrastructure:**
- Any constraints I should know about (resource limits, compliance, network)?
- Anything that's worked poorly in the past with this setup?

**Development:**
- How does local development/testing work for your team?
- Any CI/CD integration with ZenML?

**Team:**
- Solo project or team? (affects things like git hooks, naming conventions)
- Any established patterns or conventions I should follow?

**Operations:**
- How do you currently learn about pipeline failures?
- Preferences for alerting (Slack, Discord, email)?
```

Incorporate this context into your recommendations in Phase 4.

---

## Phase 4: Recommendation

### Quick Wins Catalog

| # | Quick Win | Complexity | Impact | Prerequisites |
|---|-----------|------------|--------|---------------|
| 1 | [Metadata logging](quick-wins-catalog.md#1-metadata-logging) | ⭐ | 🔥🔥🔥 | None |
| 2 | [Experiment comparison](quick-wins-catalog.md#2-experiment-comparison-zenml-pro) | ⭐ | 🔥🔥 | #1 + Pro |
| 3 | [Autologging](quick-wins-catalog.md#3-autologging-experiment-trackers) | ⭐⭐ | 🔥🔥 | Exp tracker |
| 4 | [Slack/Discord alerts](quick-wins-catalog.md#4-alerts-slackdiscord) | ⭐⭐ | 🔥🔥 | Slack/Discord |
| 5 | [Cron scheduling](quick-wins-catalog.md#5-cron-scheduling) | ⭐⭐ | 🔥🔥🔥 | Orchestrator |
| 6 | [Warm pools](quick-wins-catalog.md#6-warm-pools--persistent-resources) | ⭐ | 🔥🔥 | SageMaker/Vertex |
| 7 | [Secrets management](quick-wins-catalog.md#7-secrets-management) | ⭐⭐ | 🔥🔥🔥 | None |
| 8 | [Local smoke tests](quick-wins-catalog.md#8-local-smoke-tests) | ⭐⭐ | 🔥🔥 | Docker |
| 9 | [Tags](quick-wins-catalog.md#9-tags) | ⭐ | 🔥🔥 | None |
| 10 | [Git repo hooks](quick-wins-catalog.md#10-git-repository-hooks) | ⭐⭐ | 🔥🔥🔥 | GitHub/GitLab |
| 11 | [HTML reports](quick-wins-catalog.md#11-html-reports) | ⭐ | 🔥🔥 | None |
| 12 | [Model Control Plane](quick-wins-catalog.md#12-model-control-plane) | ⭐⭐ | 🔥🔥🔥 | None |
| 13 | [Parent Docker images](quick-wins-catalog.md#13-parent-docker-images) | ⭐⭐⭐ | 🔥🔥 | Container reg |
| 14 | [ZenML docs MCP server](quick-wins-catalog.md#14-zenml-docs-mcp-server) | ⭐ | 🔥🔥 | IDE with MCP |
| 15 | [CLI export formats](quick-wins-catalog.md#15-cli-export-formats) | ⭐ | 🔥 | None |
| 16 | [Live streaming events](quick-wins-catalog.md#16-live-streaming-events) | ⭐⭐ | 🔥🔥 | Server streaming enabled |
| 17 | [Resource pools / resource requests](quick-wins-catalog.md#17-resource-pools--resource-requests-zenml-pro) | ⭐⭐ | 🔥🔥🔥 | Pro + dynamic pipelines |

### Prioritization Matrix

**Start here (foundational):**
1. **#1 Metadata** — Foundation for everything else
2. **#9 Tags** — Organization from day one
3. **#7 Secrets** — Security baseline

**Next tier (observability):**
4. **#12 Model Control Plane** — Governance foundation
5. **#10 Git hooks** — Reproducibility
6. **#3 Autologging** or **#11 HTML reports**

**Operations tier:**
7. **#5 Scheduling** — Automation
8. **#4 Alerts** — Awareness
9. **#8 Smoke tests** — Faster iteration

**Advanced:**
10. **#6 Warm pools**, **#13 Parent images** — Performance optimization
11. **#16 Live streaming events** — Real-time progress/token updates before a step returns
12. **#17 Resource pools / requests** — Shared GPU/CPU capacity control for dynamic pipelines on ZenML Pro

**Developer experience:**
13. **#14 ZenML docs MCP** — Better IDE assistance
14. **#15 CLI export** — Scripting/automation

Use `AskUserQuestion` to ask which quick wins they want to implement based on your findings and their priorities. Present the most relevant options first based on the context gathered in previous phases.

---

## Phase 5: Prepare ⏸️

**IMPORTANT: Verify git and branch setup before making any changes.**

Before implementing any quick wins, ensure the codebase is ready for changes.

### Git Status Check

```bash
# Check current branch and status
git status
git branch --show-current

# Check for uncommitted changes
git diff --stat
```

### Branch Confirmation

Use `AskUserQuestion` to confirm branch setup:

```markdown
## 🌿 Before I make changes, let's confirm the branch setup:

**Current state:**
- Branch: `main` (or whatever branch)
- Status: [clean / X uncommitted changes]

**Questions:**
1. Should I create a feature branch for these changes? (Recommended: `feature/zenml-quick-wins`)
2. If yes, is `main` the correct base branch, or should I branch from somewhere else (e.g., `develop`)?
3. Any branch naming conventions I should follow?
```

### Create Feature Branch (if confirmed)

```bash
# Create and checkout feature branch
git checkout -b feature/zenml-quick-wins

# Or with their naming convention
git checkout -b <their-preferred-name>
```

### Why This Matters

- Prevents accidental commits to protected branches
- Allows easy rollback if something goes wrong
- Follows standard development workflow practices
- Makes it easier to review changes before merging

### Implementation Approach ⏸️

Before starting implementation, use `AskUserQuestion` to ask about the preferred approach:

```markdown
## 📝 How would you like to proceed with implementation?

**Option A: Write a plan first (Recommended)**
I can write a detailed implementation plan as a `.md` file that you can:
- Review and share with your team before changes are made
- Use as documentation of what will be implemented
- Reference during code review

After the plan is approved, I'd recommend compacting this session to start fresh with clean context for implementation.

**Option B: Proceed directly to implementation**
I'll start implementing the selected quick wins right away.

**Questions:**
1. Would you like a written plan first, or shall we proceed directly?
2. If writing a plan: where should I save it? (e.g., `docs/zenml-quick-wins-plan.md`)
```

**Why offer this choice:**
- Teams often want to review proposed changes before implementation
- A written plan serves as documentation and can be referenced in PRs
- Compacting the session before implementation clears investigation context, keeping the model focused on the task at hand
- Some users prefer to move fast; others prefer deliberate planning

---

## Phase 6: Implementation

See [quick-wins-catalog.md](quick-wins-catalog.md) for detailed implementation guides for each quick win.

### Implementation Checklist Pattern

For each quick win, follow this pattern:

```
- [ ] Verify prerequisites met
- [ ] Implement the change
- [ ] Test the implementation
- [ ] Document what was done
```

---

## Phase 7: Verification

After implementing, verify with:

```bash
# Check stack configuration
zenml stack describe

# Run a test pipeline (if applicable)
python your_pipeline.py

# Verify in dashboard (ZenML Pro)
# Check metadata, tags, model versions appear correctly
```

---

## Troubleshooting & Documentation

### When to Use Web Search

If implementation fails or documentation seems outdated:
1. Search `site:docs.zenml.io <topic>` for latest docs
2. Check specific component documentation

### Install ZenML Docs MCP Server

For better IDE assistance during implementation:

**Claude Code (VS Code):**
```bash
claude mcp add zenmldocs --transport http https://docs.zenml.io/~gitbook/mcp
```

**Cursor:**
```json
{
  "mcpServers": {
    "zenmldocs": {
      "transport": {
        "type": "http",
        "url": "https://docs.zenml.io/~gitbook/mcp"
      }
    }
  }
}
```

### Key Documentation Links

- Metadata: https://docs.zenml.io/concepts/metadata
- Tags: https://docs.zenml.io/concepts/tags
- Models: https://docs.zenml.io/concepts/models
- Secrets: https://docs.zenml.io/how-to/secrets/secrets
- Scheduling: https://docs.zenml.io/how-to/steps-pipelines/scheduling
- Alerters: https://docs.zenml.io/stacks/stack-components/alerters
- Experiment Trackers: https://docs.zenml.io/stacks/stack-components/experiment-trackers
- Code Repositories: https://docs.zenml.io/user-guides/production-guide/connect-code-repository
- Containerization: https://docs.zenml.io/how-to/containerization/containerization
- Visualizations: https://docs.zenml.io/concepts/artifacts/visualizations

---

## Quick Reference

### Minimal Metadata Example
```python
from zenml import step, log_metadata

@step
def train_model(data):
    # ... training code ...
    log_metadata({"accuracy": 0.95, "loss": 0.05, "epochs": 10})
    return model
```

### Minimal Tags Example
```python
from zenml import pipeline

@pipeline(tags=["experiment", "v2"])
def my_pipeline():
    ...
```

### Minimal Model Registration
```python
from zenml import pipeline, Model

model = Model(name="my_classifier", tags=["production"])

@pipeline(model=model)
def training_pipeline():
    ...
```
