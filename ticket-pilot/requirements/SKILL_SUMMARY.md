# Skill Summary

## Name

`agent-linear-github`

## Purpose

Coordinate Linear issue execution with GitHub-backed durable artifacts.

Linear is the task control plane:

- Issues
- Assignees
- Teams
- Labels
- Projects
- Statuses
- Progress comments

GitHub is the durable artifact store:

- Code
- Branches
- Commits
- Pull requests
- Task notes
- Deployment/test logs
- Rollback history

## Implemented Skill Files

```text
agent-linear-github/
├── SKILL.md
├── agents/
│   └── openai.yaml
├── references/
│   ├── github-conventions.md
│   ├── onboarding-settings.md
│   └── state-model.md
└── scripts/
    └── init_task_record.py
```

## Core Behavior

When the user asks Codex to work on a Linear issue, the skill tells Codex to:

1. Resolve the target Linear issue.
2. Read issue metadata, comments, labels, status, project, assignee, and links.
3. Confirm ownership before changing anything.
4. Create or use a GitHub work context.
5. Create a branch named like `linear/<issue-id>-<short-slug>`.
6. Store durable task artifacts in GitHub when useful.
7. Move Linear status according to the configured state flow.
8. Leave Linear comments at start, key nodes, failures, stop conditions, and completion.
9. Include GitHub commit/PR links back in Linear comments.

## Onboarding Behavior

The skill now has a first-run setup protocol:

1. Check local non-secret settings at:

   ```text
   ~/.codex/agent-linear-github/settings.md
   ```

2. If missing or stale, connect Linear:

   ```bash
   codex mcp login linear
   ```

   If OAuth is stale:

   ```bash
   codex mcp logout linear
   codex mcp login linear
   ```

3. Connect GitHub through a persistent auth layer:

   - Preferred: `gh auth login`
   - Also acceptable: GitHub MCP/App
   - MVP/manual fallback: one-time PAT, never stored

4. Confirm defaults with the user.
5. Save only non-secret defaults to `settings.md`.

## Credential Rules

The skill must never store or echo:

- Linear OAuth tokens
- GitHub PATs
- GitHub OAuth tokens
- GitHub App private keys or installation tokens
- Coolify/API tokens
- Client secrets
- Cookies
- Bearer tokens
- One-time OAuth callback URLs containing code/state

Credentials belong to the auth provider, not the skill.

## Current Local Status

- Source skill path:

  ```text
  /Users/luchunlan/Documents/work/周四分享会/skills/agent-linear-github
  ```

- Installed skill path:

  ```text
  /Users/luchunlan/.codex/skills/agent-linear-github
  ```

- Validation:

  ```text
  Skill is valid!
  ```

## Known Open Items

- Create the real `~/.codex/agent-linear-github/settings.md` after final onboarding confirmation.
- Decide GitHub auth method for team reuse.
- Decide whether to keep this as a skill only, wrap it as a plugin, or build a dedicated MCP.
