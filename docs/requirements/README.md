# Requirements Documents

Place your requirements documents here as markdown files.

The `/pipeline` command reads a requirements doc from this directory, breaks it into Linear stories, creates GitHub Issues, and triggers headless dev agents.

## Usage

```
/pipeline docs/requirements/my-feature.md

# Or build on an existing codebase (optional second argument)
/pipeline docs/requirements/my-feature.md https://github.com/user/starter-repo
```

## Format

Requirements docs are plain markdown. No special format required, but include:

- **What** needs to be built (features, changes, fixes)
- **Why** it's needed (context, user impact)
- **Acceptance criteria** (how to verify it's done)
- **Constraints** (performance, security, compatibility)

The `/pipeline` command will analyze the doc and break it into agent-sized stories automatically.

See [example.md](example.md) for a sample requirements doc.
