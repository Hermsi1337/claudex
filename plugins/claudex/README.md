# claudex (plugin)

The `claudex` plugin makes Claude Code an orchestrator for OpenAI Codex.

For full docs, see the [repository README](https://github.com/Hermsi1337/claudex#readme).

## Commands

- `/claudex <task>` — decompose, delegate code work to Codex, do docs in Claude, then review.
  - `--model <name>` — override Codex' model for this call. Without it, Codex' own default is used.
  - `--effort <low|medium|high|xhigh>` — force a Codex reasoning level for this call (bypasses auto-classification).
- `/claudex:setup [dir]` — verify the local `codex` CLI is installed and authenticated; with a directory argument it checks/adds the Codex trust entry for that repo instead of the current one (handy before multi-repo tasks).

After the first `/claudex` in a conversation, task-shaped follow-ups are routed through the same workflow automatically (sticky mode). Questions about the diff or codebase still get answered directly without delegating.

Without `--effort`, complex subtasks are auto-escalated to `-c model_reasoning_effort=high` (skipped if your `~/.codex/config.toml` default is already `high` or `xhigh`); standard subtasks run at your configured default. Claude never auto-lowers reasoning and never auto-escalates to `xhigh` — `low` and `xhigh` only ever happen when you ask for them via `--effort`.

Every Codex call is also launched with `-c model_verbosity=low` and a prompt rule that tells Codex to skip preamble/recap, produce a ≤5-bullet final summary, and explicitly flag unmet acceptance criteria or deviations instead of presenting a partial result as done. Claude reviews via `git diff`, so verbose Codex prose would be wasted tokens. The prompt also forbids Codex from running git inspection (`status`/`diff`/`log`), formatters, and repeated test runs — those steps belong to the orchestrator, and running them inside Codex pulls their output back into Codex' own context for no benefit. No flag to opt out yet.

Trivial mechanical edits (rename a symbol with a known target, version bump, missing import, typo fix) are kept in Claude — no Codex round-trip when one Edit call would suffice. When in doubt the work still goes to Codex.

Broad codebase research is delegated to Codex too, as read-only runs (`codex exec --sandbox read-only`) — Claude never spawns its own (expensive) subagents in claudex mode; quick lookups stay inline. Enumeration questions ("find every X") get repo orientation plus seed grep hits framed as an incomplete list to verify and extend; open investigation questions ("how/why does X happen") are posed neutrally with **no seeds and no hypothesis**, so Codex works out its own picture instead of confirming Claude's guess. The findings format includes a mandatory "Contradicts expectations" section, and read-only runs need no project-trust entry and never use the sandbox bypass.

Tasks spanning multiple repos are decomposed per repo: one Codex call per repo via `--cd <absolute-path>` (which reaches repos outside the invocation directory), run in parallel, reviewed per repo via `git -C`. The trivial-edit shortcut is disabled for multi-repo tasks and Claude never edits foreign repos itself. On macOS/Linux, trust is per repo — `/claudex:setup <dir>` adds the entry for each target, or the bypass fallback fires per repo with a notice. On native Windows the preemptive bypass applies to every write-capable call regardless of repo.

## Requirements

- OpenAI Codex CLI: `npm install -g @openai/codex`, then `codex login`.
- Claude Code on macOS, Linux, or Windows (Git Bash is supported directly; WSL works too).
- On macOS/Linux, the current project should be in your Codex trust list (`~/.codex/config.toml` → `[projects."<path>"] trust_level = "trusted"`). `/claudex:setup` checks for this and offers to add it. Without trust, `/claudex` falls back at runtime to `--dangerously-bypass-approvals-and-sandbox` for the affected call — works, but defeats the sandbox until you trust the project properly.
- On native Windows, Codex' sandbox is unavailable (Codex CLI 0.128: `workspace-write` degrades to read-only regardless of trust entries), so `/claudex` runs every write-capable Codex call with `--dangerously-bypass-approvals-and-sandbox` from the start and says so in each report. `/claudex:setup` skips the trust flow there.

The plugin uses your local Codex auth via subprocess — no API keys configured in the plugin.
