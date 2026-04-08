# Cursor Plugin for Claude Code — Requirements

**Date:** 2026-04-08
**Status:** Draft, reviewed — P0s and architectural P1s resolved. Residual P1/P2 advisory items documented. Ready for `/ce:plan`.
**Owner:** Slava Marchenko
**Scope:** Lightweight (pure-markdown Claude Code plugin)

> **Note on location:** This brainstorm concerns a personal Claude Code plugin, not a hume-mobile feature. The document lives here temporarily because the brainstorm was initiated from the hume-mobile working directory. It should be moved into the new `cursor-plugin` repo (see §9) as soon as that repo is created.

---

## 1. Problem

Claude Code currently has a first-class `codex` plugin that wraps the OpenAI Codex CLI as a delegate/rescue tool. There is **no equivalent plugin for the Cursor CLI (`cursor-agent`)**, despite cursor-agent being a very capable non-interactive coding agent with its own model pool, rules system, and worktree support.

Past ad-hoc attempts at cursor integration (`cursor-delegate` skill/agent, later renamed to `cursor-cli-runtime`) were never fully realized — the files were deleted and the functionality never landed as a coherent, installable plugin. The goal of this work is to replace that scattered history with a single, clean, minimal plugin.

The primary value is **cross-family second-opinion and hand-off**: while Claude is the parent agent, Cursor CLI can be delegated to for autonomous implementation (`/cursor:rescue`) or read-only review (`/cursor:review`) using a different model at the user's discretion.

## 2. Goals

- Ship a minimal Claude Code plugin that wraps `cursor-agent` headless mode.
- Provide **three user-invocable commands** — `/cursor:setup`, `/cursor:rescue`, `/cursor:review` — with naming that mirrors `/codex:setup` and `/codex:rescue` for consistency.
- Keep the plugin **pure markdown**: no Node scripts, no hooks, no compiled bits, no package manifests beyond Claude Code's required `plugin.json`.
- Install via marketplace (`/plugin marketplace add …`) from a dedicated git repo.
- Respect the user's own Cursor CLI configuration (model, rules, MCP, auth) — the plugin is a thin wrapper, not a config layer.

## 3. Non-Goals (explicitly OUT of v1)

The plugin deliberately excludes these features that the codex plugin ships:

### Infrastructure (runtime, hooks, state)

1. **No Node/JS runtime.** Zero `.mjs` files, zero `scripts/` directory, zero `package.json`. The entire plugin surface is markdown.
2. **No hooks.** No `hooks/hooks.json`, no `SessionStart`/`SessionEnd`/`Stop` wiring, no stop-time review gate equivalent.
3. **No async job tracking.** Every `/cursor:rescue` and `/cursor:review` call is synchronous and blocking — the subagent runs `cursor-agent -p`, waits for it to return, and passes stdout back. No job broker, no persistent state, no resume.

### Command API surface

4. **No `/cursor:status`, `/cursor:cancel`, `/cursor:result` commands.** These only make sense with async tracking.

### Customization and extensibility

5. **No cursor-specific prompting guide skill.** Users write their own prompts; the plugin does not ship opinionated prompting guidance (unlike codex, which ships `gpt-5-4-prompting`).
6. **No MCP server management.** The plugin does not wrap `cursor-agent mcp` and does not propagate `.mcp.json` or `.cursor/rules/*.mdc` — cursor-agent picks up whatever is already in the workspace.
7. **No `--model` flag.** The plugin never passes `--model` to cursor-agent. Whatever model the user has selected in their Cursor account / CLI default is the model that runs. Model choice is an out-of-band user concern.

## 4. Users and Use Cases

**Primary user:** Slava (solo at first; plugin should be structured so teammates can install it).

**Use case A — Autonomous rescue / delegation (`/cursor:rescue`).**
*"I'm stuck on this bug / I want a fresh pair of hands to fix this without me babysitting. Hand it to cursor-agent to actually edit files and run commands."*
- Runs in the current workspace.
- Direct writes to the working tree (no worktree isolation).
- Full tool access: write + shell, no approval prompts.
- Matches `codex:rescue`'s spirit: cursor finishes, user inspects `git diff`, accepts or reverts.

**Use case B — Read-only review / second opinion (`/cursor:review`).**
*"Review this diff / sanity-check this plan / give me an independent perspective from a different model family without touching the code."*
- Runs in plan mode (`--mode plan`): analysis, proposals, no edits.
- Returns a written review back to the parent context.
- Safe to run against uncommitted changes — zero blast radius.

**Use case C — First-run setup (`/cursor:setup`).**
*"Do I have cursor-agent installed? Am I logged in? If not, tell me what to run."*
- One-shot environment check.

## 5. Command Contracts

### 5.1 `/cursor:setup`

- **Purpose:** Verify the local environment can run cursor-agent.
- **Allowed tools:** `Bash(cursor-agent:*)`, `AskUserQuestion`
- **Behavior (as a slash command):**
  1. Run `command -v cursor-agent`. If missing, tell the user how to install (link to cursor.com/docs/cli) and stop.
  2. Run `cursor-agent --version` and show the version.
  3. Run `cursor-agent status`. If not authenticated, tell the user to run `cursor-agent login` in their own terminal (do **not** launch the interactive login from inside Claude Code).
  4. Report a one-line green/yellow/red status.
- **Never runs cursor-agent against a task.** Setup is diagnostic only.

> **Routing note:** `/cursor:setup` runs directly in the command turn (no subagent fork). `/cursor:rescue` forks to the **`cursor-rescue` subagent** (§6.1); `/cursor:review` forks to the separate **`cursor-review` subagent** (§6.2). Each subagent has its own locked flag set and its own inlined prompt template — there is no runtime mode switching.

### 5.2 `/cursor:rescue`

- **Purpose:** Autonomous delegation — hand a task to cursor-agent, let it edit files, return the result.
- **Context:** `fork` (subagent gets fresh context, does not inherit parent's large context).
- **Allowed tools:** `Bash`, `AskUserQuestion` *(see §6 for why the binary allowlist lives on the subagent body as prose, not on the command frontmatter)*
- **Execution model:** **Background.** Because cursor-agent autonomous runs routinely exceed the Claude Code `Bash` foreground timeout (~10 minutes), `/cursor:rescue` launches cursor-agent with `run_in_background: true` and returns immediately with the shell ID for the user to monitor via the built-in `BashOutput` tool.
- **Flow:**
  1. The command forks context and routes to the `cursor-rescue` subagent (§6.1).
  2. The subagent loads `cursor-cli-runtime` (§7.1) for its invocation contract and composes the final prompt using the template inlined in its own body.
  3. The subagent makes exactly one `Bash` call with `run_in_background: true`:
     ```
     cursor-agent -p \
       --force \
       --trust \
       --sandbox disabled \
       --workspace "$PWD" \
       --output-format text \
       -- "<composed prompt>"
     ```
  4. The subagent returns a short structured message to the parent containing: the background shell ID, a reminder to run `BashOutput <shell-id>` to stream progress, and a one-line advisory to inspect `git status` / `git diff` once cursor-agent completes.
  5. The parent agent (or user) is responsible for calling `BashOutput` to retrieve the eventual result. Result formatting when the parent reads the stream follows `cursor-result-handling` (§7.2).
- **Cursor writes directly in the current workspace.** No worktree isolation. User verifies with `git diff` / `git status`.

### 5.3 `/cursor:review`

- **Purpose:** Read-only second opinion — review a diff, plan, or area of code without touching files.
- **Context:** `fork`
- **Allowed tools:** `Bash`, `AskUserQuestion` *(see §6 for the binary-allowlist note)*
- **Execution model:** **Foreground.** Reviews are read-only and almost always complete well under the Bash foreground timeout, so `/cursor:review` runs synchronously and returns the formatted review directly.
- **Flow:**
  1. The command forks context and routes to the `cursor-review` subagent (§6.2).
  2. The subagent loads `cursor-cli-runtime` (§7.1) and composes the final prompt using the template inlined in its own body.
  3. The subagent makes exactly one foreground `Bash` call:
     ```
     cursor-agent -p \
       --mode plan \
       --trust \
       --workspace "$PWD" \
       --output-format text \
       -- "<composed prompt>"
     ```
     (`--mode plan` = read-only/planning; no `--force`, no `--sandbox disabled`.)
  4. The subagent formats cursor-agent's stdout per `cursor-result-handling` (§7.2) and returns the review to the parent.

## 6. Subagents

The plugin ships **two separate subagents**, one per mode. Each subagent has its own locked flag set and its own inlined prompt template, so there is no runtime mode switching. This eliminates a whole class of "rescue flags ran on a review request" failure modes.

**Binary-allowlist note.** Claude Code's `allowed-tools` on the command frontmatter governs only the command turn itself — once the command forks to a subagent, the subagent's own `tools:` field takes over. Bare `Bash` has no binary-level allowlist, and the plugin's "no hooks, no settings.json changes" constraints (§3) rule out the only enforcement mechanisms that could restrict which binary gets called. **The cursor subagents' bodies are therefore the only guardrail on what Bash executes.** This matches how the codex plugin's `codex-rescue` subagent works and must be explicitly acknowledged in both subagent bodies.

### 6.1 `cursor-rescue` subagent

- **Purpose:** Autonomous write-mode delegation to cursor-agent.
- **Tools:** `Bash` only. No `Read`, `Write`, `Edit`, `Grep`, `Glob`.
- **Skills declared:** `cursor-cli-runtime`, `cursor-result-handling` — these are **internal reference docs** the subagent loads for its own guidance, not externally callable helpers. The subagent still does all its work through a single `Bash` call to `cursor-agent`.
- **Execution:** Background (`run_in_background: true`). Returns the shell ID and a `BashOutput` instruction; does not wait for cursor-agent to finish.
- **Locked invocation (always, no branching):**
  ```
  cursor-agent -p --force --trust --sandbox disabled \
    --workspace "$PWD" --output-format text \
    -- "<composed prompt>"
  ```
- **Inlined prompt template** (appended to the user's task string before delegation):
  > "You are running headlessly via `cursor-agent -p` with full tool access. Fix the task below. When finished, summarize what you changed in bullet points and list any follow-ups you recommend."
- **Body contract:**
  - Single-purpose delegate — does not attempt to solve the task itself.
  - Must never alter, omit, or reorder the locked flag set above.
  - Must refuse to call any binary other than `cursor-agent`.
  - Returns cursor-agent's output verbatim per `cursor-result-handling` (§7.2).

### 6.2 `cursor-review` subagent

- **Purpose:** Read-only second-opinion review.
- **Tools:** `Bash` only.
- **Skills declared:** `cursor-cli-runtime`, `cursor-result-handling` (same reference-doc semantics as §6.1).
- **Execution:** Foreground. Waits for cursor-agent to finish, formats stdout, returns the review.
- **Locked invocation (always, no branching):**
  ```
  cursor-agent -p --mode plan --trust \
    --workspace "$PWD" --output-format text \
    -- "<composed prompt>"
  ```
  Note the absence of `--force` and `--sandbox disabled` — plan mode is read-only by construction.
- **Inlined prompt template:**
  > "You are running in plan mode. Do not edit files. Review the following and return: (1) findings, (2) recommended changes, (3) risks, (4) anything that looks wrong but you're not sure about."
- **Body contract:**
  - Single-purpose reviewer — never writes files, never runs shell.
  - Must never alter, omit, or reorder the locked flag set above.
  - Must refuse to call any binary other than `cursor-agent`.
  - Returns cursor-agent's stdout verbatim per `cursor-result-handling` (§7.2).

## 7. Skills (both pure-markdown, `user-invocable: false`)

### 7.1 `cursor-cli-runtime`

Internal reference doc describing how to invoke `cursor-agent -p`. Must cover:

**Invocation contract**
- The two locked flag sets from §6.1 and §6.2 (rescue and review), with a hard rule that they must not be modified per-call.
- How `--workspace "$PWD"` is set.
- Output format: `text` (human-readable; matches codex's verbatim-return model).

**Safe prompt composition (security-critical)**
- **Content isolation.** When the parent agent has read untrusted files (source files, PR diffs, issues, random docs), the subagent must **paraphrase or summarize** that content into the composed prompt rather than inlining it verbatim. Raw file content going into cursor-agent's prompt — which is executing under `--force --trust --sandbox disabled` — is a prompt-injection path to arbitrary file writes and shell commands.
- **The `--` separator is load-bearing.** It terminates cursor-agent's flag parser so no `-foo`-looking text inside the composed prompt can be reinterpreted as a flag. Every invocation must include it exactly as shown in §6.
- **Shell quoting.** The composed prompt is passed as a single argument to the `Bash` tool. The subagent must use a heredoc-with-sentinel pattern (e.g., `cursor-agent ... -- "$(cat <<'CURSOR_PROMPT_EOF'` … `CURSOR_PROMPT_EOF)"`) so quotes, backticks, `$(...)`, and newlines in the composed prompt cannot break out of the argument. Backing the prompt through a temp file (`mktemp` + stdin) is an acceptable alternative if the subagent body prefers it.

**Credential handling**
- `CURSOR_API_KEY` (if set in the user's environment) must **never** be logged, echoed, forwarded to the parent context, or included in error messages.
- `cursor-agent status` output may contain account identifiers — the setup command may display it, but the rescue/review subagents must not echo it to the parent context verbatim.
- Never pass `CURSOR_API_KEY` as a CLI argument or `-H` header from inside the plugin — rely on the env var or local auth store.

**Failure modes and exit-code handling**
- Non-zero exit from cursor-agent: surface the error, do not retry silently.
- Empty stdout on success: return a short explicit "cursor-agent completed with no output" line rather than an empty string.
- Authentication failure mid-session: surface the error and direct the user to `/cursor:setup`.
- Authentication expectation: assume `cursor-agent status` was green when `/cursor:setup` was last run; otherwise fail fast.

### 7.2 `cursor-result-handling`

Internal reference doc describing how to format cursor-agent output before returning to the parent context. Must cover:

- **Strip ANSI** escape sequences from stdout.
- **Preserve code blocks and diffs verbatim** — do not reflow, rewrap, or summarize them.
- **Credential scrubbing.** Before returning output, scan for anything matching common credential shapes (`CURSOR_API_KEY=…`, bearer tokens, `ssh-rsa …`, etc.) and redact them with `[REDACTED]`.
- **Rescue-mode return** (§6.1 subagent): the initial return immediately after launch is a short structured message containing: `background_shell_id`, "Run `BashOutput <id>` to stream progress", and a one-line "inspect `git diff` / `git status` once cursor-agent completes" reminder. No stdout is formatted at launch time — formatting happens when the parent later reads the stream via `BashOutput`.
- **Review-mode return** (§6.2 subagent): preserve cursor's full plan/analysis as-is, with only ANSI stripping and credential scrubbing applied.

## 8. Prompt Templates

Prompt templates are **inlined directly into each subagent body** (see §6.1 and §6.2). There is no separate `prompts/` directory — Claude Code does not auto-load `prompts/` as a plugin convention, and a pure-markdown plugin has no runtime that could `Read` template files independently.

The templates are short (~2–3 sentences each) and committed as part of the subagent `.md` files. Any future refinement happens by editing the subagent bodies directly.

## 9. Plugin Home and Distribution

- **Repo:** new personal git repo, e.g. `github.com/kharytonbatkov/cursor-plugin`
- **Structure (11 files):**
  ```
  cursor-plugin/
  ├── .claude-plugin/marketplace.json
  ├── README.md
  └── plugins/
      └── cursor/
          ├── .claude-plugin/plugin.json
          ├── commands/
          │   ├── setup.md
          │   ├── rescue.md
          │   └── review.md
          ├── agents/
          │   ├── cursor-rescue.md
          │   └── cursor-review.md
          └── skills/
              ├── cursor-cli-runtime/SKILL.md
              └── cursor-result-handling/SKILL.md
  ```
- **Install flow (README):**
  ```
  /plugin marketplace add https://github.com/kharytonbatkov/cursor-plugin
  /plugin install cursor
  /cursor:setup
  ```
- **Versioning:** semver tags on the repo; marketplace consumers get updates via `/plugin update`.

## 10. Success Criteria

The plugin is successful if all of the following are true:

1. Fresh-install flow works end-to-end: `/plugin marketplace add` → `/plugin install cursor` → `/cursor:setup` reports green.
2. `/cursor:rescue "fix the failing test in path/to/test.dart"` routes to the `cursor-rescue` subagent, launches cursor-agent in the background with `--force --trust --sandbox disabled`, returns a shell ID immediately with `BashOutput` instructions, and a later `BashOutput <id>` followed by `git diff` shows cursor's edits.
3. `/cursor:review "review my uncommitted changes"` routes to the `cursor-review` subagent, runs cursor-agent foreground with `--mode plan --trust`, returns a written review synchronously, and `git status` shows no new modifications.
4. If `cursor-agent` is missing or unauthenticated, `/cursor:setup` explains the fix and the rescue/review commands fail fast with a clear error instead of hanging.
5. All plugin files are markdown or JSON manifests. Zero `.mjs`, `.js`, `.ts`, `.sh`, `.py`, or binary files. Final file count: **11** (see §9).
6. The plugin works without any changes to `~/.claude/settings.json` (no hooks, no permission entries beyond what `allowed-tools` declares).
7. The `cursor-cli-runtime` skill spells out the content-isolation rule, the `--` separator requirement, the heredoc quoting pattern, and the `CURSOR_API_KEY` non-disclosure rule — so anyone reading the plugin understands why the invocation is shaped the way it is.

## 11. Open Questions for Planning

These are implementation-level questions that belong in `/ce:plan`, not here:

- Exact wording / YAML frontmatter of each `commands/*.md` file (including the correct Claude Code permission-matcher syntax — `Bash(cursor-agent *)` with a space vs. `Bash(cursor-agent:*)` with a colon; verify against current Claude Code permissions docs before committing).
- Exact wording of the `cursor-rescue` and `cursor-review` subagent bodies (the locked flag sets and inlined templates are fixed; the surrounding prose is not).
- Exact wording of `cursor-cli-runtime/SKILL.md` — in particular, the shell-quoting recipe (heredoc-with-sentinel vs. `mktemp`) and the credential-scrubbing regex patterns for `cursor-result-handling`.
- How `/cursor:rescue` should phrase the initial "background started, run `BashOutput <id>`" message so the parent agent reliably knows to stream it later. Consider whether the subagent should also include an advisory line about inspecting `git diff` once complete.
- `/cursor:setup` implementation detail: `command -v cursor-agent` vs. `which cursor-agent` vs. just running `cursor-agent --version` and catching the error — which combination survives the chosen permission-matcher syntax without prompting the user.
- README content and marketplace.json metadata for publishing.
- Whether a `LICENSE` (MIT?) should ship in the repo from day one.

## 12. Dependencies and Operating Assumptions

**Runtime dependencies**
- `cursor-agent` binary available on `$PATH`. (Install: `curl cursor.com/install -fsS | bash` per Cursor docs, or Homebrew if available.)
- User is authenticated to Cursor (`cursor-agent login`) with a subscription that grants CLI access.
- Claude Code is new enough to support marketplace-installed plugins with `commands/`, `agents/`, and `skills/` directories and with `run_in_background: true` on `Bash` tool calls.
- `BashOutput` is available to the parent agent for streaming background shell results. (This is a built-in Claude Code tool, not a plugin surface.)

**Operating assumptions (advisory, not enforced)**

The plugin intentionally ships without hard preconditions on workspace state — rescue mode runs wherever the user invokes it. These expectations are the user's responsibility and must be documented in the README:

- **Run `/cursor:rescue` from a trusted git repository root.** Never from `$HOME`, `/`, or any directory the user would be unhappy to see autonomously modified. Under `--force --trust --sandbox disabled`, cursor-agent has unrestricted write and shell access to `--workspace`.
- **Prefer a clean working tree before rescue.** Cursor-agent writes directly into the tree, so uncommitted work will be interleaved with cursor's edits. `git status` and `git diff` are the user's recovery surface; `git stash` or a quick commit beforehand makes them actually useful.
- **Audit `.cursor/rules/*.mdc` and `.mcp.json` in any workspace you run rescue against.** Cursor-agent auto-loads both, and a hostile rule or MCP server in the workspace silently extends cursor-agent's tool surface for the duration of the rescue run. The plugin does not inspect, warn about, or sanitize these files.
- **Treat the composed prompt as a shell argument.** The subagents isolate untrusted file contents by summarizing rather than inlining (§7.1), but the user should still avoid `/cursor:rescue` runs where the parent context has been reading untrusted material verbatim.

## 13. References

- Codex plugin (reference implementation): `~/.claude/plugins/cache/openai-codex/codex/1.0.2/`
  - Treat as structural reference only; do not copy any `.mjs`, `hooks/`, `schemas/`, or async-job infrastructure.
- `cursor-agent --help` (authoritative contract for all flags used in §5)
- Cursor headless docs: https://cursor.com/docs/cli/headless
- Prior deleted work: `cursor-delegate` / `cursor-cli-runtime` — intentionally discarded; do not resurrect.
