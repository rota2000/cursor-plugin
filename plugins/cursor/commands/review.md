---
description: Ask cursor-agent for a read-only review or second opinion in plan mode
argument-hint: "[what cursor should review]"
context: fork
allowed-tools: Bash(cursor-agent *), AskUserQuestion
---

Route this request to the `cursor:cursor-review` subagent. The final user-visible response must be cursor's review verbatim.

Raw user request:
$ARGUMENTS

## Rules

- If `$ARGUMENTS` is empty, use `AskUserQuestion` exactly once to ask the user what cursor should review. Do not invent a review target.
- The subagent runs cursor-agent with `--mode ask --trust` foreground. Ask mode is read-only by construction but allows shell commands for context gathering (git diff, etc.).
- Do not paraphrase, summarize, or add commentary before or after the subagent's output.
- After presenting the review, STOP. Do not auto-apply any suggested fixes. If the user wants changes made, they must explicitly ask.
- Do not ask the subagent to inspect files, monitor progress, or do any follow-up work.
- If cursor-agent is unavailable or unauthenticated, tell the user to run `/cursor:setup` and stop.
