# Automation System Reference

Complete technical reference for the agent pipeline template. Covers every hook, command, agent, workflow, and configuration element.

---

## System Overview

```
Human (local)                    GitHub Actions (remote)
=================                ========================

Claude Code CLI                  claude-dev.yml
  + Linear MCP        ------>      anthropics/claude-code-action@v1
  + /pipeline command              Headless Claude (dev agent)
  |                                  |
  v                                  v
Linear Project/Issues            Feature branch + PR
  |                                  |
  v                                  v
GitHub Issues (synced)           claude-review.yml
  + agent:ready label              Headless Claude (review agent)
                                     |
                                     v
                                 claude-fix.yml (if issues found)
                                   Headless Claude (fix agent)
                                   Up to 5 iterations
                                     |
                                     v
                                 Human reviews PR
                                     |
                                     v
                                 Merge to main
                                     |
                                     v
                                 linear-sync.yml
                                   Linear issue -> Done
```

Two human touchpoints: (1) write the requirements doc, (2) review and merge PRs.

---

## The Full Lifecycle

### Phase 1: Requirements to Stories

The `/pipeline` command (`.claude/commands/pipeline.md`) runs locally in the Claude Code CLI.

1. You write a requirements doc in `docs/requirements/`
2. Run `/pipeline docs/requirements/your-feature.md` (optionally add a git URL as a second argument to build on an existing codebase)
3. If a starter codebase URL was provided, Claude clones it, copies files into the project root (preserving template infrastructure), commits, and pushes
4. Claude reads the doc and connects to Linear via MCP
5. Breaks requirements into agent-sized stories (each completable in <30 turns), referencing existing files if a starter codebase was imported
6. Creates a Linear Project and Issues with structured descriptions
7. Waits for your confirmation before creating anything
8. Syncs to GitHub Issues and applies the `agent:ready` label

The `agent:ready` label is the trigger. Stories with unresolved dependencies do not get the label until their blockers are merged.

### Phase 2: Dev Agent Spawns

GitHub Issues labeled `agent:ready` trigger `claude-dev.yml`.

1. A fresh `ubuntu-latest` VM is provisioned by GitHub Actions
2. The repo is checked out, Node.js is set up, dependencies are installed
3. `anthropics/claude-code-action@v1` launches Claude Code in headless mode
4. The agent reads the GitHub Issue, reads `CLAUDE.md` for project rules
5. Implements the feature, writes tests, runs them
6. Updates issue checkboxes in real-time as it completes each requirement
7. Commits, pushes a branch, opens a PR with a `Closes #N` reference
8. The VM is destroyed. All local state is gone.

Multiple issues can trigger simultaneously. Each agent runs on its own VM with its own context window.

### Phase 3: Automated Review

The PR triggers two review systems:

**Claude Review Agent** (`claude-review.yml`):
- Runs `anthropics/claude-code-action@v1` in headless mode
- Reviews the diff against your project's review checklist (defined in the workflow)
- Checks security, architecture, test coverage
- Posts structured review comments on the PR

**CodeRabbit** (optional, external):
- 40+ linters and security scanning
- Inline code comments
- Configured via `allowed_bots` in the review workflow

If the review agent finds issues, the fix agent triggers automatically.

### Phase 4: Auto-Fix Loop

Bot review comments trigger `claude-fix.yml`.

1. A third headless Claude agent reads the review feedback
2. Fixes every issue raised
3. Runs tests and type checks
4. Pushes a commit: `fix: address review feedback [autofix 1/5]`
5. The review agent re-reviews (triggered by the new commit)
6. Loop continues up to 5 iterations
7. If still unresolved after 5 attempts, comments and stops for human help

Only bot review comments trigger the fix agent. Human comments do not trigger it, preventing unintended fix loops.

### Phase 5: Human Review and Merge

1. You review the PR (read the agent's conversation log, not just the diff)
2. Approve and merge, or leave comments for another fix cycle
3. On merge, `linear-sync.yml` fires and moves the linked Linear issue to "Done"

---

## Template Feature Inventory

### Workflows (`.github/workflows/`)

| Workflow | File | Trigger | Agent | Max Turns | Purpose |
|----------|------|---------|-------|-----------|---------|
| Dev Agent | `claude-dev.yml` | `agent:ready` label, issue assignment, `@claude` comment | Yes | 40 | Implement feature from issue |
| Review Agent | `claude-review.yml` | PR open/update, review comment, `@claude` on PR | Yes | 10 | Review PR against checklist |
| Fix Agent | `claude-fix.yml` | Bot review comments only | Yes | 15 | Fix review feedback (5 iteration cap) |
| Linear Sync | `linear-sync.yml` | PR opened, PR merged | No (shell only) | N/A | Bidirectional Linear-GitHub sync |

### Commands (`.claude/commands/`)

| Command | File | Purpose |
|---------|------|---------|
| `/pipeline` | `pipeline.md` | PM Agent: reads requirements doc, optionally imports a starter codebase, creates Linear Project and Issues, syncs to GitHub, applies `agent:ready` label |
| `/review` | `review.md` | 3-iteration QC review: (1) correctness, (2) architecture, (3) security/performance |

### Agents (`.claude/agents/`)

| Agent | File | Tools | Purpose |
|-------|------|-------|---------|
| Security Reviewer | `security-reviewer.md` | Read, Grep, Glob | OWASP-focused security review subagent |

### Hooks (`.claude/settings.json` + `.claude/hooks/`)

| Hook | Type | Matcher | What It Does |
|------|------|---------|--------------|
| Branch Protection | PreToolUse | `Edit\|Write` | Blocks all file edits on the `main` branch |
| Destructive Blocker | PreToolUse | `Bash` | Blocks force pushes, recursive deletes, database drops |

### Permissions (`.claude/settings.json`)

| Rule | Tools |
|------|-------|
| Auto-allow | Read, Glob, Grep, WebFetch, WebSearch |
| Deny | Edit/Read of `.env`, `.env.local`, `.env.production` |

### Other Configuration

| File | Purpose |
|------|---------|
| `.mcp.json` | Linear MCP server configuration (enables `/pipeline` to create Linear issues) |
| `CLAUDE.md` | Project instructions read by every agent (local and headless) |
| `.github/ISSUE_TEMPLATE/story.yml` | Structured issue template with checkboxes for agent progress tracking |
| `.gitignore` | Ignores node_modules, env files, build artifacts, test results, IDE files, tasks/ |

---

## Tool Permission Layers

Every tool call passes through a 6-layer gate system. A tool must pass all layers to execute.

```
Tool call initiated
  |
  v
1. --allowedTools whitelist (headless only)
   If the tool is not in the whitelist, BLOCKED.
   |
   v
2. permissions.deny rules (.claude/settings.json)
   If the tool + path matches a deny rule, BLOCKED.
   |
   v
3. permissions.allow rules (.claude/settings.json)
   If the tool matches an allow rule, SKIP user approval.
   |
   v
4. PreToolUse hooks (.claude/settings.json)
   Shell commands or AI prompts that run before execution.
   Exit code 2 = BLOCK the tool call.
   Exit code 0 = ALLOW the tool call.
   |
   v
5. User approval (interactive only)
   If no allow rule matched and no hook approved, prompt the user.
   In headless mode, unapproved tools are blocked.
   |
   v
6. Tool executes
   |
   v
7. PostToolUse hooks (informational only, cannot block)
```

### Layer Details

**Layer 1: `--allowedTools` whitelist** (headless agents only)

In `claude-dev.yml`, the dev agent's tool whitelist is:
```
Edit, MultiEdit, Write, Read, Glob, Grep
Bash(npm:*), Bash(npx:*), Bash(node:*), Bash(tsc:*)
Bash(mkdir:*), Bash(ls:*), Bash(cat:*), Bash(cp:*), Bash(mv:*)
Bash(chmod:*), Bash(git:*), Bash(gh:*)
```

Blocked by omission: `curl`, `wget`, `docker`, `sudo`, `ssh`, `Task`, `WebFetch`, `WebSearch`, `AskUserQuestion`.

The review agent is restricted to: `Bash(git diff:*)`, `Bash(git log:*)`, `Bash(git show:*)` (read-only).

**Layer 2: `permissions.deny`**

Prevents any agent (local or headless) from reading or modifying credential files (`.env`, `.env.local`, `.env.production`).

**Layer 3: `permissions.allow`**

Read, Glob, Grep, WebFetch, and WebSearch execute without user approval in interactive mode.

**Layer 4: PreToolUse hooks**

Two hooks are configured:

1. **Branch protection** (matches `Edit|Write`): Blocks all file edits on the main branch. Agents must work on feature branches.

2. **Destructive command blocker** (matches `Bash`): Runs `.claude/hooks/block-destructive.sh`. Strips quoted strings and heredocs from the command, then checks for recursive delete patterns, force push patterns, hard reset patterns, and database destruction patterns (SQL and MongoDB). Exit code 2 blocks the command.

**Layers 5-7**: Standard Claude Code behavior (user approval in interactive mode, execution, post-execution hooks).

---

## Hook Execution Model

### PreToolUse Hooks

- Run **before** the tool executes
- Can **block** the tool call (exit code 2)
- Can **approve** the tool call (exit code 0)
- Have a timeout (5 seconds for both hooks in this template)
- Receive tool input via environment variables
- Output is shown to the user if the hook blocks

Two types:
- `"type": "command"` - runs a shell command
- `"type": "prompt"` - sends an AI prompt (useful for semantic checks)

### PostToolUse Hooks

- Run **after** the tool executes
- **Cannot block** - informational only
- Useful for auto-formatting, notifications, or reminders

This template uses PreToolUse hooks only. Add PostToolUse hooks in `settings.json` under the `"PostToolUse"` key for project-specific needs (e.g., auto-lint after file edits).

---

## Agent Lifecycle

### Interactive (Local CLI)

```
Session start
  -> Loads CLAUDE.md, settings.json, commands
  -> Launches MCP servers (.mcp.json)
  -> Hooks fire on every tool call
  -> Subagents spawned via Task tool (own context window)
  -> Session end: MCP servers terminated, processes cleaned up
```

### Headless (GitHub Actions)

```
VM provisions (ubuntu-latest)
  -> Checkout repo
  -> Setup Node.js, npm ci
  -> anthropics/claude-code-action@v1 starts
  -> Authenticates with ANTHROPIC_API_KEY
  -> Reads CLAUDE.md + issue/PR context
  -> Runs headless (--max-turns, --allowedTools)
  -> NO MCP servers, NO commands, NO interactive prompts
  -> Hooks still active (PreToolUse fires on every tool call)
  -> VM destroyed: all state lost
```

### What's Shared vs. Isolated

| Shared (via git) | Isolated (per agent) |
|-------------------|---------------------|
| Source code | Conversation history |
| CLAUDE.md | Context window |
| .claude/settings.json | npm dependencies (fresh install) |
| .claude/hooks/ | File system state |
| .claude/agents/ | Environment variables |

Each agent gets a clean slate. No state leaks between runs.

---

## Secrets and Authentication

| Secret | Where It Lives | Used By |
|--------|---------------|---------|
| `ANTHROPIC_API_KEY` | GitHub Actions secret | claude-dev, claude-review, claude-fix workflows |
| `LINEAR_API_KEY` | GitHub Actions secret | linear-sync workflow |
| Linear OAuth token | Local OS keychain | `/pipeline` command (via Linear MCP) |
| GitHub token | Auto-provided by Actions | All workflows (via `secrets.GITHUB_TOKEN`) |
| Claude Max subscription | Local Claude Code auth | Interactive CLI usage |

The template's `.gitignore` excludes `.env`, `.env.local`, and `.env.production`. The `permissions.deny` rules block agents from reading or editing those files even if they exist.

---

## Cost and Limits

### Per-Agent Costs (Anthropic API)

| Agent | Estimated Cost | Max Turns | Timeout |
|-------|---------------|-----------|---------|
| Dev Agent | $1-5 per run | 40 | 30 min (GitHub Actions default) |
| Review Agent | $0.50-2 per run | 10 | 15 min |
| Fix Agent | $0.50-2 per run | 15 | 15 min |
| Linear Sync | $0 (no AI) | N/A | 5 min |

### Platform Costs

| Service | Cost | Notes |
|---------|------|-------|
| GitHub Actions | Free tier: 2,000 min/mo | Public repos: unlimited |
| Linear | $8/user/mo | Standard plan |
| CodeRabbit | $15/user/mo | Free for open source |
| Anthropic API | Per-token pricing | 200K context window |
| Claude Max (local) | $100-200/mo | For interactive CLI usage |

### Optimization Strategies

- Keep stories small (< 30 turns) to minimize per-run cost
- The 5-iteration fix cap prevents runaway costs from review loops
- Review agents use read-only tools (lower token usage)
- Linear sync uses zero AI (pure shell + GraphQL)

---

## Failure Modes and Mitigations

| Failure Mode | Risk | Mitigation |
|--------------|------|------------|
| **Hallucinated logic** | Agent generates plausible but incorrect code | Review agent catches logic issues. Tests must pass. Human reviews conversation log. |
| **Context window overflow** | Large codebases exhaust context, agent loses constraints | Keep stories small (< 30 turns). Turn limits cap execution. Fresh context per run. |
| **Cost explosion** | Fix-review loop runs many expensive iterations | 5-iteration fix cap. Concurrency groups. Turn limits on each agent. |
| **Error propagation** | One bad commit cascades through dependent stories | Each PR is isolated on its own branch. Branch protection requires review before merge. |
| **Security vulnerabilities** | Agent introduces injection, XSS, or OWASP issues | Security reviewer subagent. Review workflow checks for secrets and input validation. |
| **Merge conflicts** | Parallel agents modify the same files | Dependency tracking in /pipeline. Sequence stories that touch the same files. |
| **Agent stalls** | Agent gets stuck in a loop or hangs | Turn limits (40/10/15). GitHub Actions timeout (30 min). Agent shuts down when exhausted. |
| **Infinite fix loops** | Fix agent and review agent disagree endlessly | 5-iteration cap. After 5 attempts, comments on PR and stops for human intervention. |

---

## Trigger Chain

End-to-end flow showing every trigger and transition:

```
You write docs/requirements/feature.md
  |
  v
/pipeline command (local Claude Code CLI)
  |  (Optional) Clones starter codebase, copies into repo, commits, pushes
  |  Uses: Linear MCP (list_teams, create_project, create_issue)
  |  Creates: Linear Project + Issues (referencing existing code if starter was imported)
  |  Applies: agent:ready label on GitHub Issues
  |
  v
claude-dev.yml triggers (on agent:ready label)
  |  Uses: anthropics/claude-code-action@v1, --max-turns 40
  |  Creates: feature branch, commits, PR with "Closes #N"
  |  Updates: issue checkboxes in real-time
  |
  v
claude-review.yml triggers (on PR open)
  |  Uses: anthropics/claude-code-action@v1, --max-turns 10
  |  Creates: review comments on PR
  |
  v
claude-fix.yml triggers (on bot review comments)  [if issues found]
  |  Uses: anthropics/claude-code-action@v1, --max-turns 15
  |  Creates: fix commits, "fix: address review feedback [autofix N/5]"
  |  Loops: back to claude-review.yml (up to 5 iterations)
  |
  v
Human reviews PR
  |  Reads: conversation log + diff
  |  Action: approve and merge, or comment for another cycle
  |
  v
linear-sync.yml triggers (on PR merge)
  |  Uses: Linear GraphQL API (no AI)
  |  Updates: Linear issue status to "Done"
  |
  v
Deploy (your CI/CD pipeline)
```

---

## Adding Project-Specific Configuration

### Custom PreToolUse Hook (AI Prompt)

For semantic checks (e.g., "does this database query include tenant scoping?"), add a prompt-based hook to `.claude/settings.json`:

```json
{
  "matcher": "Edit|Write",
  "hooks": [
    {
      "type": "prompt",
      "prompt": "Check if this edit touches database queries. If it does, verify that every query includes tenant scoping. If a query is missing tenant scoping, respond with BLOCK and explain why."
    }
  ]
}
```

### Custom PostToolUse Hook (Auto-Lint)

Run a linter after every file edit:

```json
{
  "matcher": "Edit|Write",
  "hooks": [
    {
      "type": "command",
      "command": "npx eslint --fix \"$CLAUDE_FILE_PATH\" 2>/dev/null || true"
    }
  ]
}
```

### Custom Skill

Create `.claude/skills/your-skill/SKILL.md` with reference material that Claude loads when relevant. Skills are passive - they provide context, not commands.

### Custom Agent

Create `.claude/agents/your-agent.md` with a system prompt for a specialized subagent. Restrict tools to the minimum needed.

### Custom Review Criteria

Edit `.github/workflows/claude-review.yml` and replace the TODO comments with your project's specific review checklist.
