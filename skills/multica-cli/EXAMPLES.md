# Multica CLI Examples

## Triage an issue from an arbitrary local repo

```bash
command -v multica
multica auth status
multica workspace list
multica workspace switch PLANA
multica workspace get --output json
multica issue get MUL-123 --output json
multica issue comment list MUL-123 --output json
```

## Launch an external agent with explicit Multica context

Use your own non-reserved env names when the agent is not started by the Multica daemon:

```bash
export MCA_WORKSPACE_ID=11111111-1111-1111-1111-111111111111
export MCA_ISSUE_ID=MUL-123
```

The skill should prefer `MCA_*` only when official daemon env is absent.

## Reuse Multica's existing context file

If the unmanaged agent is launched inside a copied or mounted Multica task workdir, read:

```bash
cat .agent_context/issue_context.md
```

That file already carries the current issue ID and, for comment-triggered tasks, the triggering comment ID.

## Recover the current issue inside a daemon-managed run

Multica injects `MULTICA_WORKSPACE_ID`, `MULTICA_AGENT_ID`, and `MULTICA_TASK_ID`, but not an official `MULTICA_ISSUE_ID`.

```bash
multica agent tasks "$MULTICA_AGENT_ID" --output json
```

Then match the task whose `id` equals `$MULTICA_TASK_ID` and read its `issue_id`.

## Reply to a specific comment with multiline text

```bash
multica issue comment add MUL-123 --parent 7e947b6a-f470-4308-ad25-8538fc46cb8d --content-stdin <<'EOF'
Fixed the issue in the auth callback path.

Tests:
- `pnpm test auth`
EOF
```

## Assign an issue using canonical IDs

```bash
multica workspace members --output json
multica issue assign MUL-123 --to-id 5fb87ac7-23b5-4a7a-81fa-ed295a54545d
```

## Check execution history for an issue

```bash
multica issue runs MUL-123 --full-id --output json
multica issue run-messages 3b9609f2-7a54-42e2-a95b-cfe221e56c8d --output json
```

If you only have a short copied task prefix:

```bash
multica issue run-messages 3b9609f2 --issue MUL-123 --output json
```

## Check out the repo referenced by a workspace task

```bash
multica repo checkout https://github.com/minoo7/plana
multica repo checkout https://github.com/minoo7/plana --ref main
```

## Download an attachment for local inspection

```bash
multica attachment download 4f8e7c1f-4f8b-4f77-a7b4-4c66e93f0b9e -o /tmp/multica-downloads
```

## Prepare this machine to execute assigned tasks

```bash
multica setup
multica daemon status --output json
```

For self-hosted deployments:

```bash
multica setup self-host --server-url https://api.example.com --app-url https://app.example.com
multica daemon start --profile staging
```
