# Workflow Template — Full MVP Cycle

Validated end-to-end on 2026-06-13 with Linear issue AI-2090 and GitHub PR #1.

Each section shows the exact API calls. Adapt issue IDs, team IDs, and state IDs to the target workspace.

## Prerequisites

- `LINEAR_API_KEY` set in environment
- GitHub auth confirmed (`gh auth status` or `$GITHUB_TOKEN`)
- `~/.ticket-pilot/settings.md` exists with team/state/label info

## 1. Create Issue

```python
import subprocess, json

LINEAR_KEY = "$LINEAR_API_KEY"
TEAM_ID = "<from settings.md>"
ASSIGNEE_ID = "<from settings.md>"

query = """
mutation($input: IssueCreateInput!) {
  issueCreate(input: $input) {
    success
    issue { id identifier title url state { name } assignee { name } }
  }
}
"""

variables = {
    "input": {
        "teamId": TEAM_ID,
        "title": "#YYYYMMDD-NameN Title following team convention",
        "description": "提出日期: YYYY-MM-DD\n备注: ...\n\n## 背景\n...",
        "assigneeId": ASSIGNEE_ID,
    }
}

result = subprocess.run(
    ["curl", "-s", "-X", "POST", "https://api.linear.app/graphql",
     "-H", f"Authorization: {LINEAR_KEY}",
     "-H", "Content-Type: application/json",
     "-d", json.dumps({"query": query, "variables": variables})],
    capture_output=True, text=True
)
```

**Pitfall:** If you get `USAGE_LIMIT_EXCEEDED` with `activeIssueCount`, the workspace is at capacity. User must close/archive issues first.

## 2. Status → In Progress + Start Comment

```python
# Move to In Progress (stateId from settings.md)
linear_call('mutation { issueUpdate(id: "AI-2090", input: { stateId: "60920b17-..." }) { success issue { state { name } } } }')

# Start comment
linear_call('mutation { commentCreate(input: { issueId: "AI-2090", body: "🚀 **Starting work**\\n\\n- **Repo**: https://github.com/...\\n- **Branch**: linear/ai-2090-...\\n- **Scope**: ...\\n- **Next**: ..." }) { success } }')
```

## 3. GitHub Branch + Commit + Push

```bash
cd /path/to/repo
git checkout main && git pull origin main
git checkout -b linear/ai-2090-short-slug
git config user.name "Rachel Lu"
git config user.email "rachel.lu@feedmob.com"
# ... make changes ...
git add . && git commit -m "AI-2090 Description of work"
git push -u origin HEAD
```

## 4. Key Node Comment (sync GitHub link to Linear)

```python
body = """✅ **Key node: Work committed**

**Commit:** `3a80fb9` — [AI-2090 description](https://github.com/.../commit/HASH)
**Files:** N files, N lines
**Branch:** https://github.com/.../tree/linear/ai-2090-...

**Next:** open PR → move to In Review"""
```

## 5. Open PR + Status → In Review + Completion Comment

```bash
gh pr create --title "AI-2090: Short outcome" --body "$(cat <<'EOF'
## Linear Issue
https://linear.app/.../issue/AI-2090

## Summary
...

## Test Plan
- [x] item

Closes [AI-2090]
EOF
)" --draft
```

```python
# Move to In Review
linear_call('mutation { issueUpdate(id: "AI-2090", input: { stateId: "REVIEW_STATE_UUID" }) { success } }')

# Completion comment with PR URL
linear_call('mutation { commentCreate(input: { issueId: "AI-2090", body: "🎉 **PR opened — ready for review**\\n\\n**PR:** https://github.com/.../pull/1\\n..." }) { success } }')
```

## Comment Cadence Summary

| Point | Emoji | When | Must Include |
|-------|-------|------|-------------|
| Start | 🚀 | Before any work | repo, branch, scope |
| Key node | ✅ | After commit/milestone | commit hash, files, next step |
| Blocker | 🛑 | When stuck | error, symptom, what user must check |
| Completion | 🎉 | When PR is open | PR URL, verification, remaining |
