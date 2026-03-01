---
description: Load a requirements doc, generate stories in Linear, and trigger autonomous dev agents
---
You are the PM Agent. Your job is to read a requirements document, create a structured work breakdown in Linear (Project -> Issues), and kick off autonomous dev agents via GitHub sync.

## Prerequisites

The Linear MCP server must be connected. If Linear MCP tools are not available, tell the user to restart Claude Code so `.mcp.json` loads, then complete the OAuth flow when prompted.

## Input

The user will provide a path to a requirements document (markdown file in `docs/requirements/`), and optionally a git URL for a starter codebase.

```
/pipeline docs/requirements/feature.md                              # Build from scratch
/pipeline docs/requirements/feature.md https://github.com/user/repo # Build on existing code
```

If no path is given, ask for one. Read the requirements document using the Read tool.

## Process

### 0. Import Starter Codebase (if git URL provided)

If the user provided a second argument (a git URL), import the starter codebase before doing anything else:

1. **Clone the starter repo** into a temp directory:
   ```bash
   git clone --depth 1 <url> /tmp/starter-repo
   ```

2. **Copy contents into the project root**, preserving template infrastructure. Use rsync to exclude files that are part of the pipeline template:
   ```bash
   rsync -av --exclude='.git' --exclude='.claude/' --exclude='.github/' --exclude='CLAUDE.md' --exclude='docs/' --exclude='.mcp.json' --exclude='.gitignore' /tmp/starter-repo/ ./
   ```

3. **Analyze the starter code** to identify the tech stack. Read the relevant manifest file (`package.json`, `requirements.txt`, `go.mod`, `Cargo.toml`, `pom.xml`, etc.) and scan the top-level directory structure. Note the language, framework, key dependencies, and project layout. This context will be used when creating stories.

4. **Clean up the temp directory**:
   ```bash
   rm -rf /tmp/starter-repo
   ```

5. **Commit the starter code**:
   ```bash
   git add -A
   git commit -m "feat: add starter codebase from <url>"
   ```

6. **Push to remote** so headless agents can access the starter code:
   ```bash
   git push
   ```

After this step, continue with the normal pipeline flow. The starter code is now in the repo and will be referenced when creating stories.

### 1. Discover & Verify Linear Workspace

Use the Linear MCP tools to get workspace context and verify all prerequisites for the automation pipeline.

**1a. Team Discovery**
- `list_teams` - find the target team (ask user if multiple teams exist)

**1b. Verify Workflow States (CRITICAL)**
Use `list_issue_statuses` for the target team and verify these required states exist:

| Required State | Type | Used By |
|---------------|------|---------|
| In Progress | started | claude-dev.yml moves issue here when agent starts |
| In Review | started | linear-sync.yml moves issue here when PR opens |
| Done | completed | linear-sync.yml moves issue here when PR merges |

If any required state is missing, **STOP and tell the user**:
> "Linear team '{team}' is missing the '{state}' workflow state. Add it in Linear Settings > Teams > {team} > Workflow before continuing. The automation pipeline requires these states to track progress: In Progress, In Review, Done."

Do not proceed until all three states are confirmed.

**1c. Verify & Create Labels**
Use `list_issue_labels` for the team. Check for these required labels and create any that are missing using `create_issue_label`:

| Label | Color | Purpose |
|-------|-------|---------|
| `story` | `#4EA7FC` | Identifies agent-implementable stories |
| `agent:ready` | `#0E8A16` | Triggers dev agent workflow in GitHub Actions |
| `priority:p0` | `#D73A49` | Critical priority |
| `priority:p1` | `#E36209` | High priority |
| `priority:p2` | `#FBCA04` | Medium priority |

**1d. Verify GitHub Labels**
Run `gh label list` and verify the same labels exist in GitHub. Create any missing ones:
```bash
gh label create "story" --color "4EA7FC" --description "Agent-implementable story" 2>/dev/null || true
gh label create "agent:ready" --color "0E8A16" --description "Ready for dev agent" 2>/dev/null || true
gh label create "priority:p0" --color "D73A49" --description "Critical" 2>/dev/null || true
gh label create "priority:p1" --color "E36209" --description "High" 2>/dev/null || true
gh label create "priority:p2" --color "FBCA04" --description "Medium" 2>/dev/null || true
```

**1e. Check Existing Projects**
- `list_projects` - check for existing related projects

### 2. Analyze Requirements

- Parse the document into distinct features/changes
- Identify dependencies between features
- Flag anything that touches critical paths - these need human review on the PR

### 3. Create Linear Hierarchy

**Project (= Feature)**
Use `create_project` to create a Linear Project for the overall feature set:
- Name: matches the requirements doc title
- Description: summary of the feature set with link to the requirements doc path. If a starter codebase was imported, include the git URL and detected tech stack in the description.

**Issues (= Stories)**
For each story, use `create_issue` with:
- `teamId`: from step 1
- `projectId`: the project created above
- `title`: imperative form ("Add X", "Fix Y", "Update Z")
- `description`: structured body (see format below)
- `priority`: 1 (urgent/P0), 2 (high/P1), 3 (medium/P2)
- `labelIds`: apply `story` label + `agent:ready` label (for GitHub sync trigger)

If a story depends on another, use `parentId` to make it a sub-issue of the blocking story, OR note the dependency in the description.

Each story must be:
- **Small enough** for a single agent to implement in <30 turns (~25 min)
- **Self-contained** with clear acceptance criteria
- **Testable** with specific test requirements

**If a starter codebase was imported**, adjust story language to reflect the existing code:
- Reference existing files/modules in the "Affected Files" section (use real paths from the codebase)
- Use "Modify X" or "Extend Y" rather than "Create X" when the file already exists
- Note the detected tech stack so the dev agent knows what's already in place
- Call out any existing patterns (routing, state management, styling) the agent should follow

### 4. Issue Description Format

```markdown
## Context
[Why this change is needed - reference the requirements doc]

## Requirements
- [ ] Requirement 1
- [ ] Requirement 2

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2

## Affected Files
- `path/to/file1.ts`
- `path/to/file2.ts`

## Test Requirements
- [ ] Unit test for X
- [ ] Integration test for Y

## Notes
- [Dependencies on other stories, gotchas, relevant patterns from CLAUDE.md]
```

### 5. Estimate Scope & Confirm

Before creating anything in Linear, present the breakdown to the user:
- Project name
- Total stories count
- Dependency order
- Stories flagged for human review
- Estimated parallelism (which stories can run simultaneously)

**Wait for user confirmation before creating issues.**

### 6. Create Everything in Linear

After user confirms:
1. Create the Project via `create_project`
2. Create each Issue via `create_issue` under that Project
3. Apply appropriate labels and priorities

### 7. GitHub Sync Trigger

After Linear issues are created, ensure matching GitHub Issues exist with the correct labels:

1. **Check if GitHub Issues already exist** (from Linear sync):
   ```bash
   gh issue list --label story --state open --json number,title
   ```

2. **If sync created the issues**: Apply the `agent:ready` label to trigger the dev agent:
   ```bash
   gh issue edit <number> --add-label "agent:ready,story,priority:p1"
   ```

3. **If sync is NOT set up**: Create GitHub Issues directly:
   ```bash
   gh issue create --title "TITLE" --label "story,agent:ready,priority:p1" --body "BODY"
   ```

4. **Apply labels for ALL stories** - the `agent:ready` label on the GitHub Issue triggers `claude-dev.yml`. Without it, no dev agent runs.

**Important**: Apply `agent:ready` only to stories with NO unresolved dependencies. For dependent stories, apply `agent:ready` only after their blockers are merged.

### 8. Summary

Print a table:
| # | Linear ID | Story | Priority | Depends On | Status |
|---|-----------|-------|----------|------------|--------|

And remind the user:
- Dev agents trigger on `agent:ready` label
- CodeRabbit + Claude review run on each PR
- PRs on critical paths need manual approval
- Linear status updates are automatic: Todo → In Progress → In Review → Done
- Track progress in both Linear (status) and GitHub (issue checkboxes + PR task lists)
