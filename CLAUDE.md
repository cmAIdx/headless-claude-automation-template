# CLAUDE.md

## Project Overview

<!-- TODO: Describe your project in 2-3 sentences. What it does, what stack, where it's deployed. -->

## Critical Policies

<!-- TODO: List non-negotiable rules that every agent (human or AI) must follow. -->
<!-- These get enforced by hooks in .claude/settings.json but should also be stated here. -->

## Architecture

<!-- TODO: Directory layout, database schema, import conventions, key dependencies. -->

## Commands

<!-- TODO: Dev server, build, lint, test commands for your project. -->

## Testing

<!-- TODO: Test framework, how to run tests, coverage requirements. -->

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
