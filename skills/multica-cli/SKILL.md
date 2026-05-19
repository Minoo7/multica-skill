---
name: multica-cli
description: Uses the `multica` CLI from any directory to authenticate, select workspaces, inspect issues/comments/runs, post updates, download attachments, and check out linked repos without relying on a Multica-generated task worktree. Use when the user mentions Multica, `multica` commands, workspace issues/comments/agents/autopilots, or needs an external agent to interact with a Multica workspace safely.
---

# Multica CLI

## Quick start

- Confirm the CLI is present and authenticated: `command -v multica && multica auth status`
- If needed, log in with `multica login` or `multica login --token=`
- Detect runtime context in this order:
  - official daemon env: `MULTICA_WORKSPACE_ID`, `MULTICA_TASK_ID`, `MULTICA_AGENT_ID`
  - task-local `.agent_context/issue_context.md` if present
  - caller-supplied fallback env for non-Multica runs: `MCA_WORKSPACE_ID`, `MCA_ISSUE_ID`
  - task-local `AGENTS.md` as a secondary source
- Discover workspace context with `multica workspace list`
- If multiple workspaces are available, switch the active one with `multica workspace switch <workspace>`
- Prefer `--output json` for read commands
- Use the CLI only for Multica data; do not `curl` Multica URLs directly
- Start `multica daemon start` only when this machine must execute assigned agent work

## Workflow

### 1. Establish context outside a Multica worktree

- Do not assume the current directory matters; Multica context comes from auth + active workspace
- In daemon-managed runs, trust `MULTICA_WORKSPACE_ID` as the current workspace
- Do not expect an official `MULTICA_ISSUE_ID` env; prefer `.agent_context/issue_context.md`, then caller env, then task-local `AGENTS.md`, then agent-task metadata
- Load workspace details with `multica workspace get --output json`
- If the user provides an issue key or ID, fetch it with `multica issue get <id> --output json`
- For comment-driven work, inspect the thread with `multica issue comment list <issue-id> --output json`

### 2. Read before acting

- Use `--full-id` when you need canonical UUIDs for follow-up commands
- Resolve people and agents before assigning or mentioning: `multica workspace members --output json` and `multica agent list --output json`
- For long or multiline text, prefer `--content-stdin`, `--content-file`, `--description-stdin`, or `--description-file`

### 3. Common write actions

- Comment: `multica issue comment add <issue-id> ...`
- Update issue fields: `multica issue update <id> ...`
- Change status only: `multica issue status <id> <status>`
- Assign precisely: `multica issue assign <id> --to-id <uuid>`

### 4. Repo and artifact work

- If the issue references a codebase, fetch it with `multica repo checkout <url> [--ref <branch-or-sha>]`
- Download platform attachments with `multica attachment download <attachment-id>`
- Treat checked-out repos as normal local git work after checkout

### 5. Mentions have side effects

- `mention://issue/...` links are safe references
- `mention://member/...` notifies a human
- `mention://agent/...` enqueues another agent run
- Default to no member or agent mentions unless escalation or delegation is intentional

## Reference

See [REFERENCE.md](REFERENCE.md) for the command map and [EXAMPLES.md](EXAMPLES.md) for copyable flows.
