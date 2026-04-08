---
description: Check whether the local Cursor CLI is installed and authenticated
allowed-tools: Bash(command -v *), Bash(cursor-agent *), AskUserQuestion
---

Check whether cursor-agent is installed and authenticated. Report a one-line status.

## Steps

1. Run `command -v cursor-agent`. If exit code is non-zero, tell the user:
   - cursor-agent is not installed.
   - Install with: `curl cursor.com/install -fsS | bash` (see https://cursor.com/docs/cli).
   - Stop here. Do not proceed to further steps.

2. Run `cursor-agent --version`. Display the version to the user.

3. Run `cursor-agent status`. Based on the output:
   - If authenticated: note "authenticated" (do **not** echo the raw `cursor-agent status` output verbatim — it may contain account identifiers).
   - If not authenticated: tell the user to run `cursor-agent login` in their own terminal. Do **not** launch the interactive login from inside Claude Code.
   - If error: surface the error message.

4. Report a one-line summary:
   - **Green:** cursor-agent installed and authenticated. Ready to use `/cursor:rescue` and `/cursor:review`.
   - **Yellow:** cursor-agent installed but not authenticated. Run `cursor-agent login` first.
   - **Red:** cursor-agent not found. Install it first.

## Rules

- This command is diagnostic only. Never run cursor-agent against a task.
- Never echo `CURSOR_API_KEY` or raw account details.
