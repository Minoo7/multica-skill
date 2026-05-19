# Multica CLI Reference

This skill is for agents that need to use Multica even when they are not running inside a Multica-generated task directory.

## Core rule

Use the `multica` CLI for all Multica platform interactions. Do not use `curl`, `wget`, or direct HTTP calls to Multica resource URLs.

## Operating model

The upstream docs show that authentication, profiles, watched workspaces, and the active workspace live in Multica's CLI config rather than in the current repository. In practice, that means an agent can operate from any local directory as long as:

- `multica` is installed
- the CLI is authenticated
- the correct workspace is active

Start the daemon only if the task needs this machine to act as a Multica runtime. Reading issues, posting comments, switching workspaces, and checking execution history are CLI operations and do not require a checked-out Multica task worktree.

## Dynamic context model

There are two distinct modes:

- Daemon-managed task: Multica starts the agent process and injects official runtime env such as `MULTICA_WORKSPACE_ID`, `MULTICA_TASK_ID`, `MULTICA_AGENT_ID`, `MULTICA_AGENT_NAME`, `MULTICA_SERVER_URL`, and `MULTICA_TOKEN`.
- External/manual run: the agent is launched outside Multica's daemon. In that case, no official task env exists unless the caller sets its own wrapper env.

Recommended resolution order for this skill:

1. Official daemon env from Multica
2. Task-local `.agent_context/issue_context.md`
3. Caller-supplied fallback env for this skill, using a non-reserved prefix such as `MCA_WORKSPACE_ID` and `MCA_ISSUE_ID`
4. Task-local `AGENTS.md`
5. Explicit user-provided IDs in the prompt
5. Interactive CLI discovery such as `multica workspace list` and `multica issue list --output json`

Use a non-reserved fallback prefix like `MCA_*` for manual launches. Do not depend on `custom_env` to set `MULTICA_*` keys: upstream docs state that keys starting with `MULTICA_*` are ignored by the daemon.

## What Multica injects today

From the current upstream daemon code, Multica injects these official task env vars for daemon-managed runs:

- `MULTICA_WORKSPACE_ID`
- `MULTICA_TASK_ID`
- `MULTICA_AGENT_ID`
- `MULTICA_AGENT_NAME`
- `MULTICA_SERVER_URL`
- `MULTICA_TOKEN`
- `MULTICA_DAEMON_PORT`
- `MULTICA_TASK_SLOT`
- optionally `MULTICA_AUTOPILOT_ID`, `MULTICA_AUTOPILOT_RUN_ID`, `MULTICA_QUICK_CREATE_TASK_ID`

Important limitation: the daemon task struct contains an `issue_id`, but the daemon does not currently inject an official `MULTICA_ISSUE_ID` env var. Do not assume it exists.

## Existing on-disk context files

Multica already writes portable context files into the task workdir:

- `.agent_context/issue_context.md`
- `AGENTS.md` for most providers, `CLAUDE.md` for Claude, `GEMINI.md` for Gemini
- `.multica/project/resources.json` when project resources exist

The current upstream implementation writes `.agent_context/issue_context.md` with human-readable task metadata including:

- issue ID
- trigger type
- triggering comment ID when applicable
- quick-start command

For unmanaged agents, this markdown file is the best current pickup point because it already exists and does not depend on the daemon process staying alive.

## First-pass checklist

1. Confirm availability: `command -v multica`
2. Confirm auth: `multica auth status`
3. If not logged in:
   - Browser flow: `multica login`
   - Headless flow: `multica login --token=`
4. Discover context: `multica workspace list`
5. If needed, switch workspace: `multica workspace switch <workspace>`
6. Verify active workspace: `multica workspace get --output json`

## How to resolve the current issue dynamically

Preferred order:

1. If `.agent_context/issue_context.md` exists, read it first.
2. If the caller provided `MCA_ISSUE_ID`, use it.
3. If a Multica-generated `AGENTS.md` exists, read the issue ID and triggering comment ID from it.
4. If you are in a daemon-managed task and have `MULTICA_AGENT_ID` plus `MULTICA_TASK_ID`, use `multica agent tasks "$MULTICA_AGENT_ID" --output json` and match the current task ID to recover its `issue_id`.
5. If none of the above exists, ask the user for the issue ID or discover candidate issues with `multica issue list --output json`.

This is the practical answer to "can issue context be an env var?":

- Yes, but only if Multica itself starts injecting it, or if your external launcher sets its own fallback env such as `MCA_ISSUE_ID`.
- No, not via agent `custom_env` under the `MULTICA_*` namespace, because the daemon ignores those keys.

If you want a more machine-readable contract for unmanaged agents, the next upstream improvement would be a JSON sibling such as `.multica/task/context.json`. Until then, use `.agent_context/issue_context.md` as the canonical portable file.

## Install or update

If the CLI is missing or too old for a documented command:

- Install with Homebrew: `brew install multica-ai/tap/multica`
- Upgrade Homebrew installs: `brew upgrade multica-ai/tap/multica`
- Upgrade script or manual installs: `multica update`

## Read commands

Prefer `--output json` when another tool or agent will read the result.

- Workspace
  - `multica workspace list`
  - `multica workspace get --output json`
  - `multica workspace members [workspace-id] --output json`
  - `multica workspace watch <workspace-id>`
  - `multica workspace unwatch <workspace-id>`
- Agents
  - `multica agent list --output json`
- Issues
  - `multica issue list [--status X] [--priority X] [--assignee X] [--assignee-id <uuid>] [--project <id>] [--limit N] [--full-id] [--output json]`
  - `multica issue get <id> --output json`
  - `multica issue comment list <issue-id> [--since <RFC3339>] --output json`
  - `multica issue label list <issue-id> --output json`
  - `multica issue subscriber list <issue-id> --output json`
  - `multica issue runs <issue-id> [--full-id] --output json`
  - `multica issue run-messages <task-id> [--issue <issue-id>] [--since <seq>] --output json`
- Labels
  - `multica label list --output json`
- Projects
  - `multica project list [--status X] [--output json]`
  - `multica project get <id> --output json`
  - `multica project resource list <project-id> --output json`
- Repos and attachments
  - `multica repo checkout <url> [--ref <branch-or-sha>]`
  - `multica attachment download <attachment-id> [-o <dir>]`
- Autopilots
  - `multica autopilot list [--status X] [--full-id] [--output json]`
  - `multica autopilot get <id> --output json`
  - `multica autopilot runs <id> [--limit N] --output json`

## Write commands

- Issues
  - `multica issue create --title "..." [--description "..."] [--description-stdin] [--description-file <path>] [--priority X] [--status X] [--assignee X|--assignee-id <uuid>] [--parent <issue-id>] [--project <project-id>] [--due-date <RFC3339>] [--attachment <path>]`
  - `multica issue update <id> [--title X] [--description X] [--description-stdin] [--description-file <path>] [--priority X] [--status X] [--assignee X|--assignee-id <uuid>] [--parent <issue-id>] [--project <project-id>] [--due-date <RFC3339>]`
  - `multica issue status <id> <status>`
  - `multica issue assign <id> --to <name>|--to-id <uuid>`
- Comments
  - `multica issue comment add <issue-id> [--content "..." | --content-stdin | --content-file <path>] [--parent <comment-id>] [--attachment <path>]`
  - `multica issue comment delete <comment-id>`
- Labels
  - `multica label create --name "..." --color "#hex"`
  - `multica issue label add <issue-id> <label-id>`
  - `multica issue label remove <issue-id> <label-id>`
- Subscribers
  - `multica issue subscriber add <issue-id> [--user <name>|--user-id <uuid>]`
  - `multica issue subscriber remove <issue-id> [--user <name>|--user-id <uuid>]`
- Projects
  - `multica project create --title "..." [--description "..."] [--status X] [--icon "..."] [--lead "..."]`
  - `multica project update <id> [--title X] [--description X] [--status X] [--icon "..."] [--lead "..."]`
  - `multica project status <id> <status>`
- Autopilots
  - `multica autopilot create --title "..." --agent <name> --mode create_issue|run_only [--description "..."]`
  - `multica autopilot update <id> [--title X] [--description X] [--status active|paused] [--mode create_issue|run_only]`
  - `multica autopilot trigger <id>`
  - `multica autopilot delete <id>`

## ID handling

- Human table output may show issue keys such as `MUL-123` and short UUID prefixes.
- Use `--full-id` when you need canonical UUIDs for scripting.
- Prefer `--assignee-id` and `--to-id` when names overlap.
- For `run-messages`, a copied short task ID must be scoped with `--issue <issue-id>`.

## Text input modes

Use the shortest safe input mode:

- `--content "..."` or `--description "..."` for short single-line text
- `--content-stdin` or `--description-stdin` for multiline text via heredoc
- `--content-file <path>` or `--description-file <path>` for exact UTF-8 file content, especially in shell environments that may re-encode non-ASCII

## Comment and mention safety

Multica-generated task directories may include a task-specific `AGENTS.md` that tells the agent which issue and triggering comment to reply to. When such a snippet is present:

- treat issue IDs, comment IDs, repo URLs, and reply rules as task-local context
- reply to the triggering comment with `--parent <comment-id>` when instructed
- do not change issue status unless the task explicitly asks for it

Mention links are not just formatting:

- `[MUL-123](mention://issue/<issue-id>)` is a safe issue reference
- `[@Name](mention://member/<user-id>)` notifies a human
- `[@Name](mention://agent/<agent-id>)` enqueues a new agent run

Default to no member or agent mention unless the user explicitly wants a loop-in, escalation, or new delegation.

## Daemon and runtime notes

Use these when the task is about making the current machine available as a runtime:

- `multica setup`
- `multica setup self-host`
- `multica daemon start`
- `multica daemon start --foreground`
- `multica daemon status --output json`
- `multica daemon logs`
- `multica daemon stop`

High-signal environment variables from the upstream docs:

- `MULTICA_WORKSPACES_ROOT`
- `MULTICA_DAEMON_POLL_INTERVAL`
- `MULTICA_DAEMON_HEARTBEAT_INTERVAL`
- `MULTICA_DAEMON_MAX_CONCURRENT_TASKS`
- `MULTICA_AGENT_TIMEOUT`
- `MULTICA_CODEX_MODEL`
- `MULTICA_CODEX_ARGS`

## Failure handling

- If auth fails, stop and fix login before issuing workspace or issue commands.
- If the workspace is wrong, switch it before reading or writing issue state.
- If the CLI lacks a needed operation, do not invent an HTTP workaround.
- If you are in a daemon-managed task, never fall back to a stale user-global workspace config when `MULTICA_WORKSPACE_ID` is missing; treat that as broken runtime context.
- If you need issue context and no `MCA_ISSUE_ID` or task-local `AGENTS.md` exists, recover it from `multica agent tasks "$MULTICA_AGENT_ID" --output json` before acting.
- If a documented command is missing locally, update the CLI before assuming the workflow is unsupported.
- If a command returns ambiguous human-formatted output, rerun it with `--output json` or `--full-id`.
