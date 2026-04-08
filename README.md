# Cursor Plugin for Claude Code

A minimal, pure-markdown Claude Code plugin that wraps `cursor-agent` in headless mode. Delegate autonomous coding tasks (`/cursor:rescue`) or get read-only second opinions (`/cursor:review`) from a different model family without leaving Claude Code.

## Install

```
/plugin marketplace add <repo-url>
/plugin install cursor
/cursor:setup
```

## Commands

### `/cursor:setup`

Check whether `cursor-agent` is installed and authenticated. Reports green/yellow/red status. Does not run any tasks.

### `/cursor:rescue [task]`

Delegate an autonomous coding task to cursor-agent. Cursor-agent runs in the background with full write and shell access (`--force --trust --sandbox disabled`). Returns a background shell ID immediately — use `BashOutput <id>` to stream progress.

After cursor-agent finishes, inspect changes with `git diff` / `git status`.

### `/cursor:review [task]`

Ask cursor-agent for a read-only second opinion in plan mode (`--mode plan`). Runs foreground and returns the review synchronously. Does not edit files.

## Safety posture

`/cursor:rescue` runs cursor-agent with `--force --trust --sandbox disabled`. This combination disables all approval prompts and gives cursor-agent unrestricted write and shell access to the current workspace. This is intentional — the value of rescue mode is autonomous, unattended execution.

### Operating assumptions (your responsibility)

- **Run `/cursor:rescue` from a trusted git repository root.** Never from `$HOME`, `/`, or any directory you would be unhappy to see autonomously modified.
- **Prefer a clean working tree before rescue.** Cursor-agent writes directly into the tree, so uncommitted work will be interleaved with cursor's edits. `git stash` or a quick commit beforehand makes `git diff` / `git reset` actually useful for recovery.
- **Audit `.cursor/rules/*.mdc` and `.mcp.json` in your workspace.** Cursor-agent auto-loads both. A hostile rule or MCP server silently extends cursor-agent's tool surface for the duration of the rescue run.
- **The composed prompt is a shell argument.** The subagents isolate untrusted file content by summarizing rather than inlining, but avoid `/cursor:rescue` runs where the parent context has been reading untrusted material verbatim.

## Model selection

The plugin never passes `--model` to cursor-agent. Whatever model you have selected in your Cursor account or CLI default is the model that runs. Model choice is an out-of-band concern managed via `cursor-agent` settings.

## Dependencies

- `cursor-agent` on `$PATH` — install via `curl cursor.com/install -fsS | bash`
- Authenticated to Cursor — run `cursor-agent login`
- Claude Code with marketplace plugin support

## References

- [Cursor headless CLI docs](https://cursor.com/docs/cli/headless)
- `cursor-agent --help` for the full flag reference

## License

MIT
