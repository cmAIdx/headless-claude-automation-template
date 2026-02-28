# Agent Pipeline Template

A reusable template for autonomous software delivery using headless Claude agents. Feed a requirements doc in, review pull requests out. Two human touchpoints.

This template contains the complete automation infrastructure -- GitHub Actions workflows, Claude Code configuration, Linear integration, and safety hooks -- extracted from a production pipeline and stripped of all project-specific code. Fork it, fill in your CLAUDE.md, and you have a working agent pipeline.

## What's in the box

```
.github/workflows/
  claude-dev.yml          Dev agent: reads GitHub Issue, implements, opens PR
  claude-review.yml       Review agent: auto-reviews every PR on open/update
  claude-fix.yml          Fix agent: reads bot review feedback, pushes fixes (5 iteration cap)
  linear-sync.yml         Bidirectional Linear <> GitHub sync

.github/ISSUE_TEMPLATE/
  story.yml               Structured issue template with checkboxes for agent progress tracking

.claude/
  settings.json           Safety hooks + permission rules
  hooks/block-destructive.sh
  commands/pipeline.md    /pipeline command (requirements doc -> Linear stories -> GitHub Issues)
  commands/review.md      /review command (3-pass QC review)
  agents/security-reviewer.md

.mcp.json                 Linear MCP server config
CLAUDE.md                 Project instructions skeleton (fill this in)
docs/requirements/        Where requirements docs live
```

## How the automation works

The pipeline turns a markdown requirements document into deployed code with two human touchpoints: writing the doc and reviewing PRs.

### The flow

```
You write a requirements doc (docs/requirements/feature.md)
  |
  v
/pipeline command (Claude Code CLI)
  - Reads the requirements doc
  - Breaks it into sized stories using Linear MCP
  - Creates Linear issues with structured descriptions
  - Syncs to GitHub Issues
  - Applies the "agent:ready" label
  |
  v
claude-dev.yml triggers (GitHub Actions)
  - anthropics/claude-code-action@v1 spins up a headless Claude agent
  - Agent reads the GitHub Issue
  - Agent checks out the repo, reads CLAUDE.md for project rules
  - Implements the feature, writes tests, runs them
  - Updates issue checkboxes in real time as it works
  - Commits, pushes a branch, opens a PR with a "Closes #N" reference
  - Agent shuts down. The runner is gone.
  |
  v
claude-review.yml triggers (on PR open)
  - A separate headless Claude agent reviews the diff
  - Checks security, architecture, test coverage
  - Posts structured review comments on the PR
  |
  v
claude-fix.yml triggers (on bot review comments)
  - A third headless Claude agent reads the review feedback
  - Fixes every issue raised
  - Pushes a commit: "fix: address review feedback [autofix 1/5]"
  - Review agent re-reviews (triggered by the new commit)
  - Loop continues up to 5 iterations
  - If still unresolved after 5 attempts, comments and stops for human help
  |
  v
You review the PR
  - Read the agent's conversation log (every decision is visible)
  - Approve and merge, or leave comments for another fix cycle
  |
  v
linear-sync.yml fires (on PR merge)
  - Moves the linked Linear issue to "Done"
  - The story is complete
```

### Key design decisions

**Why GitHub Actions, not a persistent agent?** No infrastructure to maintain. Each agent is a fresh GitHub Actions runner that spins up, does work, and self-destructs. No VMs, no pm2, no process monitoring. The workflow YAML is the entire deployment.

**Why separate dev/review/fix agents?** Each agent gets a fresh context window scoped to its job. The dev agent focuses on implementation. The review agent focuses on finding problems. The fix agent focuses on addressing feedback. No context pollution between roles.

**Why real-time checkbox updates?** The dev agent updates issue checkboxes as it completes each requirement, not in a batch at the end. You can watch progress live in the GitHub Issue while the agent works.

**Why a 5-iteration fix cap?** Prevents infinite loops where the fix agent and review agent disagree. After 5 attempts, a human needs to intervene. In practice, most issues resolve in 1-2 iterations.

### Safety mechanisms

- **Branch protection hook**: Blocks all file edits on the `main` branch. Agents must work on feature branches.
- **Destructive command blocker**: Catches force pushes, recursive deletes, and database drop commands before they execute.
- **Credential deny rules**: Blocks reading or editing `.env`, `.env.local`, `.env.production`.
- **Bot-only fix trigger**: The fix agent only responds to bot review comments, not human ones. Prevents unintended fix loops.
- **Concurrency groups**: Only one fix agent runs per PR at a time.
- **Turn limits**: Dev agent capped at 40 turns, review at 10, fix at 15.

## Quick start

See [SETUP.md](SETUP.md) for the full 9-step setup guide. The short version:

1. Use this template to create a new repo (or copy the files into an existing one)
2. Add `ANTHROPIC_API_KEY` and `LINEAR_API_KEY` to GitHub repo secrets
3. Run `/install-github-app` in Claude Code to install the Anthropic GitHub App
4. Fill in `CLAUDE.md` with your project's architecture, policies, and commands
5. Write a requirements doc in `docs/requirements/`
6. Run `/pipeline docs/requirements/your-feature.md`
7. Watch the agents work. Review the PRs.

## What you customize per project

| File | What to change |
|------|---------------|
| `CLAUDE.md` | Everything. This is the source of truth every agent reads. |
| `claude-review.yml` | Replace TODO comments with your project's review criteria |
| `claude-dev.yml` | Adjust `--max-turns`, `--allowedTools`, test/build commands |
| `story.yml` | Add critical-path checkboxes for your project's sensitive areas |
| `.claude/settings.json` | Add project-specific hooks (data isolation, lint, etc.) |

## Links

### Core tools
- [anthropics/claude-code-action](https://github.com/anthropics/claude-code-action) -- the GitHub Action that runs Claude Code headless in CI
- [Claude Code documentation](https://docs.anthropic.com/en/docs/claude-code)
- [Linear MCP server](https://linear.app/docs/mcp) -- how /pipeline creates stories
- [CodeRabbit](https://www.coderabbit.ai/) -- optional automated code review (40+ linters)

### Background reading
- [Anthropic: Agentic Coding Trends 2026](https://resources.anthropic.com/2026-agentic-coding-trends-report) -- industry data on autonomous agents
- [Anthropic: Claude Code Agent Teams](https://docs.anthropic.com/en/docs/claude-code/agent-teams) -- multi-agent patterns
- [Autonomous Coding Quickstart](https://github.com/anthropics/claude-quickstarts/tree/main/autonomous-coding) -- Anthropic's reference implementation
- [Addy Osmani: AI Coding Workflow](https://addyosmani.com/blog/ai-coding-workflow/) -- spec-first approach from Google Chrome lead

### Alternative patterns
- [Cyrus](https://www.atcyrus.com/) -- Linear-native agent (alternative to GitHub Actions approach)
- [Devin](https://devin.ai/) -- fully managed autonomous agent
- [GitHub Copilot Coding Agent](https://docs.github.com/en/copilot/using-github-copilot/using-copilot-coding-agent-to-work-on-tasks/about-assigning-tasks-to-copilot) -- GitHub's native agent
