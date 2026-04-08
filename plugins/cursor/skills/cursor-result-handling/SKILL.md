---
name: cursor-result-handling
description: Internal guidance for formatting cursor-agent output before returning to the parent context
user-invocable: false
---

# Cursor Result Handling

When cursor-agent returns output, apply these formatting rules before returning to the parent context.

## ANSI Stripping

Strip all ANSI escape sequences from stdout. Cursor-agent's text output may include terminal color codes that render as garbage in Claude Code's context.

## Preserve Code Verbatim

Preserve code blocks, diffs, file paths, and line numbers exactly as cursor-agent reports them. Do not reflow, rewrap, summarize, or edit the content.

## Credential Scrubbing

Before returning output, scan for anything matching common credential shapes and redact with `[REDACTED]`:

- `CURSOR_API_KEY=...` or any `*_API_KEY=...` pattern
- `Bearer ...` tokens (redact everything after "Bearer ")
- `-----BEGIN ... KEY-----` blocks through `-----END ... KEY-----`
- `ssh-rsa ...` or `ssh-ed25519 ...` key material
- `ghp_`, `gho_`, `github_pat_` prefixed tokens

When in doubt about whether something is a credential, redact it. False positives are far less harmful than leaked secrets.

## Rescue-Mode Return (cursor-rescue subagent)

The cursor-rescue subagent launches cursor-agent with `run_in_background: true` and returns **immediately** — it does not wait for cursor-agent to finish.

The initial return message must contain exactly:
1. The background shell ID (from the Bash tool's response)
2. An instruction: "Run `BashOutput <shell-id>` to stream cursor-agent's progress."
3. A one-line advisory: "Once cursor-agent completes, inspect changes with `git status` / `git diff`."

Return nothing else. No commentary, no predictions about what cursor-agent will do.

Stdout formatting (ANSI strip, credential scrubbing) happens later — when the parent agent or user reads the stream via `BashOutput`.

## Review-Mode Return (cursor-review subagent)

The cursor-review subagent runs cursor-agent foreground and waits for completion.

Apply ANSI stripping and credential scrubbing to cursor-agent's stdout, then return the review verbatim. Preserve cursor's full analysis, findings, and recommendations as-is.

## Critical Guardrail

After presenting review findings, **STOP**. Do not auto-apply fixes. Do not suggest you are about to make changes. If the user wants changes made, they must explicitly ask. Auto-applying fixes from a review is strictly forbidden, even if the fix looks obvious.

If cursor-agent was never successfully invoked (e.g., binary missing, auth failure), do not generate a substitute answer. Report the failure and stop.

If cursor-agent reports that setup or authentication is required, direct the user to `/cursor:setup` and do not improvise alternate auth flows.
