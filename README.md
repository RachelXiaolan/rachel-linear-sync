# Agent Linear GitHub Test

Test repo for the `agent-linear-github` skill MVP.

## Purpose

This repo is used to validate the Linear ↔ GitHub sync workflow:
- Linear issues trigger GitHub branches
- Work is tracked via Linear comments
- PRs link back to Linear issues

## Structure

```
tasks/
  <ISSUE-ID>/
    task.json
    notes.md
    logs/
```
