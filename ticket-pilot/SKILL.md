---
name: ticket-pilot
description: >
  Ticket Pilot: coordinate Linear issue work with GitHub-backed task artifacts.
  Works with any agent (Hermes, Codex, Claude Code, etc.). Use when the user asks to
  work on a Linear issue, sync Linear and GitHub, update Linear comments/status while
  coding, inspect teammate issue status, create branches/PRs for Linear tasks, or run
  an agent workflow across Linear issues/projects and GitHub repositories.
version: 2.1.0
author: Rachel Lu
license: MIT
platforms: [linux, macos, windows]
prerequisites:
  env_vars: [LINEAR_API_KEY]
  commands: [curl]
metadata:
  hermes:
    tags: [Linear, GitHub, Issues, Project Management, Automation, Workflow]
---

# Ticket Pilot

## Overview

Run a Linear issue as a traceable engineering workflow.

- **Linear** = task control plane: issue status, assignee, project, labels, progress comments.
- **GitHub** = durable artifact store: code, branches, commits, PRs, task notes, logs.
- **This skill** = orchestrator: connects both sides, keeps them in sync.

This skill is **agent-agnostic**. It does not depend on any specific MCP server or CLI
framework. It works with whatever auth and tools the agent's environment provides.

## Companion Skills (load for platform mechanics)

This skill defines the **orchestration workflow**. For the underlying platform mechanics,
load these companion skills:

- **`linear`** — Linear GraphQL API patterns + Python CLI helper (`scripts/linear_api.py`).
  Use `linear_api.py` for faster one-liners: `python3 linear_api.py whoami`, `list-teams`,
  `create-issue`, `update-status`, `add-comment`, `search-issues`, etc. Prefer this helper
  over hand-crafted curl for all standard Linear operations.
- **`github-repo-management`** — repo creation, cloning, settings, releases, secrets.
- **`github-pr-workflow`** — branch → commit → PR → CI monitoring → merge lifecycle.
- **`github-issues`** — GitHub issue management (when GitHub issues are also in scope).

The curl/GraphQL examples below are for reference. For any standard Linear operation,
check the `linear` skill's CLI helper first — it wraps the common cases.

## Authentication

### Linear

Use a **Linear Personal API key** (preferred for simplicity):

1. Go to Linear → Settings → Account → Security & access → Personal API keys
   (https://linear.app/settings/account/security)
2. Create a key, copy it.
3. Set it as `LINEAR_API_KEY` environment variable (or in agent env config).

API details:
- Endpoint: `https://api.linear.app/graphql` (POST)
- Header: `Authorization: $LINEAR_API_KEY` (no "Bearer" prefix)
- All requests are POST with `Content-Type: application/json`
- Both UUIDs and short IDs (e.g. `AI-1972`) work

Quick test:
```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ viewer { id name email } }"}' | python3 -m json.tool
```

If the agent environment has a Linear MCP available, that works too — use the MCP
tools directly and skip the API key.

### GitHub

**Option 1 — gh CLI (preferred):**
```bash
gh auth login          # interactive setup
gh auth status         # verify
```

**Option 2 — Personal Access Token (PAT):**
Set `GITHUB_TOKEN` in the environment. The skill will use it for API calls.
```
GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxx
```

**Option 3 — GitHub App / MCP:**
If a GitHub MCP or App is configured, use it directly.

The skill auto-detects auth method:
```bash
# Detection priority
if command -v gh &>/dev/null && gh auth status &>/dev/null; then
  AUTH="gh"        # use gh CLI for everything
elif [ -n "$GITHUB_TOKEN" ]; then
  AUTH="token"     # use curl + GITHUB_TOKEN
else
  AUTH="none"      # ask user to authenticate
fi
```

## Onboarding

Before running issue work, check for local non-secret settings:

```text
~/.ticket-pilot/settings.md
```

If the file is missing, incomplete, or the user asks to change defaults, run onboarding:

1. **Verify Linear access.** Test the API key with a `viewer` query. If it fails, ask
   the user to create a new key at https://linear.app/settings/account/security.
2. **Verify GitHub access.** Run `gh auth status` or check `$GITHUB_TOKEN`. If neither
   is available, present both options (gh CLI or PAT) and let the user choose.
3. **Discover Linear context.** Query the user's workspace, teams, workflow states,
   projects, and labels with read-only calls.
4. **Confirm defaults with the user.** Show discovered values and ask for confirmation
   on: workspace, default team, user identity, status mapping, GitHub owner/org,
   default repo or repo policy, branch naming, and comment cadence.
5. **Save only non-secret defaults** to `settings.md`. Never save tokens, keys, or secrets.

Read `references/onboarding-settings.md` for the exact settings format and revalidation rules.

## Label & Assignee Rules

When creating or updating Linear issues, always apply labels and assignees explicitly:

- **Default assignee = current user** (from `settings.md`). Auto-assign to the user unless they say otherwise.
- **Default label = user's personal label** (e.g. `Rachel`). Always attach the user's label to their own issues.
- **If the issue involves other teammates** — ask the user which labels and assignees to add. Do not guess.
- **If the issue is ambiguous** (could belong to multiple people) — ask one concise question rather than auto-assigning.
- **Never remove or change someone else's label/assignee** without asking.

Summary: **Default to the user, ask when uncertain.**

## Start Workflow

1. **Resolve the target Linear issue** from the user request.
   - Accept issue IDs like `AI-2090`, Linear URLs, or a query like "my current issue".
   - Read the issue: title, description, status, assignee, team, project, labels,
     comments, and linked PRs.
   - Read team workflow states before changing anything.

2. **Confirm ownership and scope.**
   - Apply Label & Assignee Rules (above).
   - If unassigned and the user wants it, assign it.
   - If another human owns it, ask before reassigning.
   - If ambiguous, ask one concise question.

3. **Create a linked GitHub issue** (if one doesn't exist yet).
   - Use the same title (with Linear ID prefix, e.g. `AI-2090: <title>`).
   - Body must include the Linear issue URL and a summary.
   - This GitHub issue mirrors progress comments from Linear (see "Comment Sync" below).
   - GitHub issues **cannot** track status (In Progress etc.) or priority — Linear is
     the source of truth for those. Do not try to manage status/priority on GitHub.
   - Link the GitHub issue number back to Linear via a comment.

4. **Create a GitHub work context.**
   - Use or create a repository selected by the user or inferred from settings/issue links.
   - Create or switch to a branch named `linear/<issue-id>-<short-slug>`.
   - Optionally create a task artifact folder with `scripts/init_task_record.py`.

5. **Mark work started.**
   - Move the Linear issue to "In Progress" (or team equivalent).
   - Leave a **comment on both Linear and GitHub issue** with branch/repo, scope, and first planned step.

## During Work

Leave comments at every meaningful boundary. Do not save everything for one final update.
**Each comment goes to both Linear and the linked GitHub issue.**

**Required comment points:**

| Point | Content |
|------|---------|
| **Start** | "Starting work", repo/branch, scope, first step |
| **Key node** | tests pass/fail, deploy attempt, external blocker, decision made |
| **Failure** | observed error, log source, retry count, next step |
| **Stop** | exact UI or human action needed |
| **Completion** | PR/commit links, verification, remaining risks |

## Comment Sync (Linear ↔ GitHub Issue)

Every Linear progress comment must be **mirrored to the linked GitHub issue**. This keeps
both platforms in sync for team members who check either one.

**Linear → GitHub comment mirroring:**

```bash
# After posting a comment to Linear, also post to the GitHub issue
# Linear comment includes: progress text + GitHub links
# GitHub comment includes: same progress text + Linear issue link

gh issue comment <GH_ISSUE_NUMBER> \
  --repo "$OWNER/$REPO" \
  --body "🚀 **Starting work**

- Branch: \`linear/ai-2090-ticket-pilot-skill\`
- Linear: https://linear.app/feedmob/issue/AI-2090
- Scope: MVP workflow validation

Next: create branch, commit skill files, open PR."
```

**What GitHub issues CAN mirror:**
- Comments / progress updates ✓
- Links to commits, branches, PRs ✓
- Labels (can use GitHub labels for basic categorization) ✓

**What GitHub issues CANNOT mirror (Linear is source of truth):**
- Workflow status (Backlog → In Progress → In Review → Done) — GitHub has no equivalent
- Priority levels — GitHub has no built-in priority field
- Team/project association — GitHub has no team/project equivalent
- Assignee changes — sync Linear assignee as primary; GitHub assignee optional

### Linear Comment (use `linear` skill CLI helper)

```bash
# Preferred: use the linear skill's Python CLI
SCRIPT=$(find ~/.hermes -path '*skills/productivity/linear/scripts/linear_api.py' 2>/dev/null | head -1)
python3 "$SCRIPT" add-comment AI-1972 --body "Starting work. Branch: linear/ai-1972-add-login. Next: review auth flow."
```

### Linear Status Update (use `linear` skill CLI helper)

```bash
# Preferred: use the linear skill's Python CLI
python3 "$SCRIPT" update-status AI-1972 --state "In Progress"
# Or by stateId (UUID):
python3 "$SCRIPT" update-status AI-1972 --state-id "<STATE_UUID>"
```

> **Note:** The `linear` skill CLI handles state name → stateId resolution automatically.
> For raw GraphQL or edge cases not covered by the CLI, fall back to curl patterns
> documented in the `linear` skill.

## State Flow

Default state movement:

```text
Backlog/Todo → In Progress → In Review → Done
```

- Use `Done` only when the user requested direct completion and verification passed.
- Use `In Review` when a PR or human check is expected.
- Do not change state if: the issue belongs to another assignee without consent,
  required credentials/decisions are missing, or verification could not run.

Read `references/state-model.md` for status mapping and custom team states.

## GitHub Sync

GitHub plays two roles in this workflow:

### 1. GitHub Issue (progress mirror)
- Created alongside each Linear issue
- Mirrors all progress comments from Linear
- GitHub issue **cannot** track status/priority — that stays on Linear
- Keeps GitHub-only team members in the loop

### 2. GitHub Artifacts (durable storage)
- code and config changes
- generated files for deployment
- task notes and decision logs
- test/deploy logs
- PR review history
- rollback points

Include the Linear issue ID in branch names, commit messages, PR titles, and PR bodies.

Read `references/github-conventions.md` for the Linear-specific branch/commit/PR conventions.
For the full GitHub mechanics (clone, create repo, push, create PR, monitor CI, merge),
load the `github-repo-management` and `github-pr-workflow` skills.

## Team Visibility

When the user asks to see coworker status:
- List Linear users and teams.
- Query issues by assignee, team, status, label, project, and updated time.
- Summarize by person: active, blocked, recently done, stale.
- Do not modify coworker-owned issues unless explicitly requested.

## Credential Rules

- Ask for credentials only at the moment they are needed.
- Treat tokens as one-time secrets. Never store in files, git, comments, logs, or commits.
- Do not print tokens back to the user.
- Store stable non-secret defaults (user IDs, workspace/team IDs, org/repo names,
  branch conventions) only after explicit confirmation.
- Reuse saved defaults silently on future runs, but revalidate when auth fails or
  the user says defaults changed.

## Pitfalls

### Linear free plan — active issue limit

Linear free-tier workspaces have a cap on **active issues**. When the limit is reached, `issueCreate` returns:

```json
{
  "errors": [{
    "extensions": {
      "code": "USAGE_LIMIT_EXCEEDED",
      "meta": { "usageMetric": "activeIssueCount" }
    }
  }]
}
```

**Before creating issues during onboarding or MVP demos**, check the workspace is not at capacity. If it is, the user must archive/close existing issues or upgrade the plan. Do not retry `issueCreate` after this error — it will keep failing until the workspace is under quota.

### Settings auto-discovery

During onboarding, **proactively query** the workspace to auto-fill settings rather than asking the user to type IDs. Query in this order:

1. `viewer { id name email }` — confirm identity
2. `teams { nodes { id name key } }` — get team keys/IDs
3. `workflowStates(filter: { team: { key: { eq: "KEY" } } })` — get status UUIDs for the default team
4. `issueLabels(first: 30)` — discover available labels
5. `projects(first: 20)` — list projects

Present the discovered values and let the user confirm rather than blank-prompting for each one.

## Stop Conditions

Stop and ask the user when:
- A required UI-only action is needed.
- A credential is needed.
- A deploy target cannot see the repo.
- A service remains unhealthy after two retries.
- A permission error changes the trust model.
- Continuing would reassign someone else's issue or publish private material.

When stopping, leave a Linear comment if Linear is available:

```text
Blocked at <step>. Symptom: <observed>. Error: <message>. Need you to check <specific thing>.
```

## Auth Detection

GitHub auth is auto-detected at runtime. The skill should check in this priority order:

```bash
if command -v gh &>/dev/null && gh auth status &>/dev/null; then
  AUTH="gh"        # use gh CLI for everything
elif [ -n "$GITHUB_TOKEN" ]; then
  AUTH="token"     # use curl + GITHUB_TOKEN
else
  AUTH="none"      # present options to user: gh CLI or PAT
fi
```

When `AUTH="none"`, offer both paths and let the user choose — do not force one method.

## Team Naming Conventions

When creating issues, always follow the target team's existing naming pattern. Before creating your first issue on a team, query recent issues to learn the pattern.

FeedMob ken-team (AI) pattern (discovered 2026-06-13):

```text
#YYYYMMDD-NameN 描述内容
```

Examples:
- `#20260613-RachelLu-1 Ticket Pilot — Linear issue 自动同步更新的标准化 skill`
- `#20260612-Leo03 YouTube调控，每天发一篇最有价值的`
- `#20260610-Windy02 要把Rachel Lu 加进来会议说工作情况`

Description body starts with:
```text
提出日期: YYYY-MM-DD
备注: 来源：... Action（... 值日）。
```

For other teams or workspaces, **always sample 10+ recent issues first** to detect the pattern before creating.

## Full Workflow Checklist

The end-to-end cycle. Each step must complete before the next:

1. **Create Linear issue** (or resolve existing) — follow team naming convention, apply user's label + assignee
2. **Create linked GitHub issue** — same title with Linear ID, body links back to Linear
3. **Status → In Progress** — query workflow states first to get stateId
4. **Start comment** — post to **both** Linear and GitHub issue (repo/branch, scope, first step)
5. **Create GitHub branch** — `linear/<issue-id>-<slug>` from latest main
6. **Do work + commit** — include issue ID in commit message
7. **Key node comment** — post to **both** platforms (commit hash, file list, what's done)
8. **Push branch + open PR** — draft by default, include Linear issue link in body
9. **Status → In Review** — Linear only (GitHub has no status)
10. **Completion comment** — post to **both** platforms (PR URL, verification, remaining steps)

See `references/workflow-template.md` for the full MVP transcript with exact API calls.

## Resources

- `scripts/init_task_record.py` — create a task artifact folder for a Linear issue.
- `references/onboarding-settings.md` — first-run onboarding and settings format.
- `references/state-model.md` — status mapping and comment rules.
- `references/github-conventions.md` — branch, commit, PR, and task artifact conventions.
- `references/workflow-template.md` — validated full-cycle MVP workflow with exact API calls.
