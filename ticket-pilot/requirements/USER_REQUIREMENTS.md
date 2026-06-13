# User Requirements

## Goal

Build a reusable Codex workflow so Rachel and teammates can work with Linear issues inside Codex while keeping Linear, GitHub, and agent progress synchronized.

The desired loop is:

```text
Codex agent <-> Linear <-> GitHub
```

## Primary Scenario

Rachel installs the skill in Codex.

On first use, Codex should:

1. Ask Rachel to connect Linear.
   - Prefer an automatic browser OAuth popup.
   - If automatic popup fails, show a login URL.
   - After authorization, Codex/Linear MCP should remember the login until it expires.

2. Ask Rachel to connect GitHub.
   - Preferred long-term path: GitHub App, GitHub MCP, or `gh auth login`.
   - MVP path: ask for a GitHub access token only when needed.
   - Tokens must not be saved into markdown, code, git remotes, comments, logs, or commits.

3. Confirm default settings through conversation.
   - Linear workspace
   - Linear user name/email
   - Linear default team
   - Linear status mapping
   - GitHub org/owner
   - Default repo or ask-per-issue policy
   - Branch naming
   - Comment/status sync behavior

4. Save only non-secret defaults to a local markdown file.

After onboarding, Codex should use these defaults automatically unless:

- Login expires
- Workspace/repo access fails
- Rachel asks to change defaults
- A requested write action conflicts with saved defaults

## Expected Capabilities

### Linear Read

Codex should be able to read:

- Issues
- Projects or initiatives
- Teams
- Labels
- Statuses
- Assignees
- Comments
- Teammate issue status

### Linear Write

Codex should be able to:

- Assign an issue to Rachel when requested
- Move issue status, such as Backlog/Todo -> In Progress -> In Review -> Done
- Leave progress comments
- Add links to GitHub branches, commits, PRs, and task artifacts

### GitHub Sync

Codex should be able to:

- Create or use a repository
- Create a branch for a Linear issue
- Commit code and task artifacts
- Create PRs
- Store task notes, generated code, logs, and rollback points
- Link GitHub work back to Linear comments

## Progress Comment Contract

For issue work, Codex should comment in Linear at each meaningful point:

- Start
- Key node
- Failure
- Stop/blocker
- Completion

Codex should not save everything for one final update.

## Stop Conditions

Codex should stop and ask Rachel when:

- A credential is needed
- A UI-only action is required
- The target workspace/team/repo is ambiguous
- A write would affect another person's issue
- A deployment or integration remains unhealthy after repeated attempts
- Continuing would expose private material or change the trust model

## Authentication Principle

The skill must not become a credential manager.

Correct division of responsibility:

```text
settings.md: stores non-secret defaults
Codex MCP: stores Linear OAuth login state
gh / GitHub App / GitHub MCP: stores GitHub auth state
skill: orchestrates workflow and asks for login when auth is missing or expired
```

## Recommended Onboarding Defaults

These are proposed defaults, not yet confirmed as final settings:

```text
Linear workspace: feedmob
Linear user: Rachel lu
Linear email: rachel.lu@feedmob.com
Default team: ken-team
Default label: Rachel
Start status: In Progress
Review status: In Review
Complete status: Done

GitHub owner/org: feed-mob
Default repository: ask
Repository visibility: ask
Branch prefix: linear/
Task artifact path: .linear/tasks/
PR policy: draft
Auto-move status on start: yes
Auto-move status on completion: In Review
```

## Future Packaging Options

### Skill Only

Fastest path. Good for documenting workflow and making Codex behave consistently.

### Plugin

Better for team distribution. Can bundle skill, MCP config, helper scripts, and UI-facing metadata.

### Dedicated MCP

Best for robust reuse. Can expose stable tools such as:

- `onboard_user`
- `get_my_linear_context`
- `start_issue_work`
- `sync_issue_progress`
- `create_task_artifact`
- `link_github_pr`
- `summarize_teammate_status`

The MCP can enforce credential boundaries and settings validation more reliably than a skill alone.
