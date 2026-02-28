# CLAUDE.md

<!-- IMPORTANT: Keep this file concise. Every line is loaded into every agent session
     (local and headless). Bloated instructions get ignored. For each line, ask:
     "Would removing this cause an agent to make a mistake?" If not, cut it.
     Move verbose reference material to .claude/skills/ (loaded on demand). -->

## Project Overview

<!-- What this project does in 1-2 sentences. Tech stack. Deployment target. -->
<!-- Example: "SaaS platform for X (Next.js 15, TypeScript, PostgreSQL). Deployed on Vercel." -->

## Critical Policies

<!-- Non-negotiable rules. Use BOLD for constraints that, if violated, cause real damage.
     These should also be enforced by hooks in .claude/settings.json, but state them here
     so agents understand WHY the hooks exist. -->

<!-- Examples:
- **Every database query MUST include tenant scoping (org_id filter). Violation = data breach.**
- **Never use mock data. This is production. Only real API responses.**
- **Never commit secrets. Use environment variables.**
-->

## Architecture

<!-- Directory layout (just the key directories, not every file).
     Import conventions (aliases, module boundaries).
     Key dependencies agents need to know about. -->

<!-- Example:
```
src/
  app/       # Next.js App Router pages + API routes
  lib/       # Core business logic
  components/# React components
  types/     # TypeScript types
```
- Import alias: `@/` maps to `src/`
- API responses: use `apiSuccess(data)`, `apiError(msg)` from `@/utils/api-response`
-->

## Commands

<!-- Build, test, lint, dev server. Only include commands agents can't guess. -->

```bash
# npm run dev          # Dev server
# npm run build        # Production build
# npm run lint         # Linter
# npm test             # Full test suite
# npx tsc --noEmit     # Type check
```

## Testing

<!-- Test framework, how to run tests, coverage requirements.
     Be specific: agents need to know exactly what to run before opening a PR. -->

<!-- Example:
- Framework: Jest (unit/integration), Playwright (UI)
- New features require tests. Never reduce coverage unless deleting the code under test.
- Run `npm test` before every PR.
-->

## Headless Agent Guidelines

<!-- These rules apply when agents run autonomously in GitHub Actions
     via anthropics/claude-code-action@v1. They have no human to ask. -->

### Verification Before PR

Every agent MUST verify before creating a pull request:
1. Tests pass: `npm test`
2. Types clean: `npx tsc --noEmit`
3. Lint clean: `npm run lint`
4. Build succeeds: `npm run build`

If any check fails, fix the issue. If it cannot be fixed after 2 attempts, stop and report the failure in a PR comment. Do NOT create a PR with failing checks.

### Autonomy Boundaries

Agents operate autonomously within these boundaries:

**Proceed without asking:**
- Implement features described in the GitHub Issue
- Write and run tests
- Fix lint errors and type errors
- Create feature branches and open PRs
- Update issue checkboxes to track progress

**Stop and escalate (comment on the issue/PR):**
- Ambiguous or contradictory requirements
- Changes that affect authentication, authorization, or encryption
- Database schema migrations or destructive data changes
- Architectural decisions not covered by this file
- Tests that fail for reasons unrelated to the current change

### Progress Tracking

Dev agents MUST update GitHub Issue checkboxes as they work. Update 3-5 times during implementation, not in a single batch at the end. This lets humans monitor progress in real time.

### Commit Conventions

- One logical change per commit
- Commit messages: imperative mood, concise (`Add user validation`, not `Added some validation stuff`)
- PR title matches the issue title
- PR body includes `Closes #N` to auto-link the GitHub Issue

### Context Management

- Read CLAUDE.md at session start (this happens automatically)
- Stay focused on the assigned issue. Do not refactor unrelated code.
- If context is running low, prioritize: (1) completing the current task, (2) running tests, (3) opening the PR

## Working Style

### Task Management
1. Write plan to `tasks/todo.md` with checkable items
2. Check in before starting implementation
3. Mark items complete as you go; add review section when done

### Non-Trivial Changes
For API integrations, schema changes, auth/security, core workflows:
1. **Impact & Risk**: Identify affected areas/dependencies. Call out security, data integrity, performance risks. If uncertain, STOP and ask.
2. **Design & Approach**: Describe proposed solution. Confirm alignment with existing patterns.
Do NOT write code until both steps are complete. If something goes sideways, STOP and re-plan.

### Autonomous Bug Fixing
When given a bug report: just fix it. Point at logs, errors, failing tests -- then resolve them. Zero hand-holding required.

### Self-Improvement Loop
After ANY correction: update `tasks/lessons.md` with the pattern. Review lessons at session start.

## Custom Triggers

These are natural language phrases typed in conversation (not registered skills):

- **`qc`**: 3-iteration deep review -- (1) correctness & completeness, (2) architecture & tech debt, (3) security, performance, production readiness
- **`runit`**: Execute all remaining steps without pausing. Only stop when fully implemented, tested, and production ready.
- **`name <label>`**: Set the terminal tab title to the given label so the user can identify this agent session. Run: `echo -ne "\033]0;<label>\007"` via Bash. Confirm with: "Session named: <label>". **VS Code prerequisite** (one-time): add `"terminal.integrated.tabs.title": "${sequence}"` to VS Code settings -- without this, VS Code ignores title escape sequences.

## Compaction
When compacting, always preserve: the full list of modified files, any test commands that need to run, and the current task plan.
