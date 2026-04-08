---
description: Delegate an autonomous coding task to cursor-agent in background mode
argument-hint: "[what cursor should do]"
context: fork
allowed-tools: Bash(cursor-agent *), AskUserQuestion
---

Route this request to the `cursor:cursor-rescue` subagent. The final user-visible response must be the subagent's output verbatim (the background shell ID and BashOutput instructions).

Raw user request:
$ARGUMENTS

## Rules

- If `$ARGUMENTS` is empty, use `AskUserQuestion` exactly once to ask the user what cursor should do. Do not invent a task.
- The subagent is a thin forwarder only. It uses one `Bash` call with `run_in_background: true` to invoke cursor-agent and returns the shell ID plus `BashOutput` instructions.
- Do not paraphrase, summarize, or add commentary before or after the subagent's output.
- Do not ask the subagent to inspect files, monitor progress, poll status, fetch results, or do any follow-up work.
- If cursor-agent is unavailable or unauthenticated, tell the user to run `/cursor:setup` and stop.
