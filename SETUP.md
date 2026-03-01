# Pipeline Setup Guide

How to set up the autonomous agent pipeline for a new project.

## Prerequisites

- GitHub repository
- [Claude Code CLI](https://claude.ai/code) installed locally
- [Linear](https://linear.app) workspace
- Anthropic API key (for headless agents)

## Step 1: Repository Setup

If you used this as a GitHub template, you're already done with the file structure. Otherwise, copy these directories into your existing repo:

```
.github/           # Workflows, issue templates
.claude/           # Hooks, commands, agents, settings
.mcp.json          # Linear MCP config
CLAUDE.md          # Project instructions (fill this in!)
docs/requirements/ # Where requirements docs live
```

## Step 2: GitHub Secrets

Go to **Settings > Secrets and variables > Actions** and add:

| Secret | Value | Used By |
|--------|-------|---------|
| `ANTHROPIC_API_KEY` | Your Anthropic API key | claude-dev, claude-review, claude-fix workflows |
| `LINEAR_API_KEY` | Your Linear personal API key | linear-sync workflow |

## Step 3: Claude Code GitHub App

In your local Claude Code terminal:

```bash
/install-github-app
```

This installs the Anthropic GitHub App on your repo, which `anthropics/claude-code-action@v1` requires.

## Step 4: Linear MCP (for /pipeline)

The `.mcp.json` file configures the Linear MCP server. On first use, Claude Code will open an OAuth flow in your browser to authenticate with Linear.

Test it:
```bash
# In Claude Code CLI
/pipeline docs/requirements/your-feature.md

# Or build on an existing codebase (optional second argument)
/pipeline docs/requirements/your-feature.md https://github.com/user/starter-repo
```

## Step 5: Fill in CLAUDE.md

The template `CLAUDE.md` has TODO markers. Fill in:
- Project overview (what it does, tech stack)
- Critical policies (security rules, data handling)
- Architecture (directory layout, conventions)
- Commands (dev, build, test, lint)
- Testing (framework, coverage requirements)

This file is read by every agent (local and headless). It is the single source of truth for how to work in your codebase.

## Step 6: Customize Review Criteria

Edit `.github/workflows/claude-review.yml` and replace the TODO comments with your project's review checklist. Examples:
- Data isolation rules
- No mock data policy
- Compliance requirements
- Critical paths requiring human review

## Step 7: Branch Protection (Recommended)

Configure GitHub branch protection on `main`:

1. **Settings > Branches > Add rule** for `main`
2. Enable **Require a pull request before merging**
3. Enable **Require approvals** (set to 1)
4. Enable **Require status checks to pass** - add: `claude-review`
5. (Optional) **Dismiss stale approvals when new commits are pushed**

## Step 8: CodeRabbit (Optional)

Install [CodeRabbit](https://coderabbit.ai) on your GitHub repo for automated code review with 40+ linters and security scanning. The review workflow is configured to accept CodeRabbit's comments (`allowed_bots: "coderabbitai,claude"`).

## Step 9: Global Settings (Per Machine)

These are user-level settings, not per-project. Set them once on each machine:

### `~/.claude/settings.json`
```json
{
  "enableAllProjectMcpServers": false,
  "permissions": {
    "deny": [
      "Edit(~/.ssh/**)",
      "Edit(~/.gnupg/**)",
      "Edit(~/.aws/**)",
      "Edit(~/.config/gh/**)",
      "Edit(~/.git-credentials)",
      "Read(~/.ssh/id_*)",
      "Read(~/.gnupg/**)"
    ]
  }
}
```

### GitHub MCP Server
```bash
claude mcp add github
```

## How It Works

```
/pipeline docs/requirements/feature.md [optional: https://github.com/user/starter-repo]
  |
  v  [If starter URL provided: clone, copy into repo, commit, push]
Linear Issues created --> GitHub Issues synced (linear-sync.yml)
  |
  v  [agent:ready label triggers claude-dev.yml]
Dev Agent implements feature --> creates PR
  |
  v  [PR triggers claude-review.yml]
Review Agent posts feedback
  |
  v  [Bot review triggers claude-fix.yml]
Fix Agent reads feedback --> pushes fixes --> up to 5 iterations
  |
  v  [All checks pass]
Human reviews PR --> merges to main
  |
  v
Linear issues auto-close with status "Done"
```

**Human touchpoints**: (1) write the requirements doc, (2) review and merge PRs.

## File Reference

```
.github/
  workflows/
    claude-dev.yml        # Dev agent: issue -> implementation -> PR
    claude-review.yml     # Review agent: auto-review PRs
    claude-fix.yml        # Fix agent: auto-fix bot review feedback (5 iteration cap)
    linear-sync.yml       # PR -> Linear issue sync (create on open, close on merge)
  ISSUE_TEMPLATE/
    story.yml             # Structured issue template for dev agents

.claude/
  settings.json           # Hooks (branch protection, destructive blocker) + permissions
  hooks/
    block-destructive.sh  # Blocks dangerous commands (see script for full pattern list)
  commands/
    pipeline.md           # /pipeline - requirements -> Linear -> agents
    review.md             # /review - 3-iteration QC review
  agents/
    security-reviewer.md  # Security review subagent

.mcp.json                 # Linear MCP server config
CLAUDE.md                 # Project instructions (fill in!)
docs/requirements/        # Requirements docs for /pipeline
```
