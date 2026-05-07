# claudex (plugin)

The `claudex` plugin makes Claude Code an orchestrator for OpenAI Codex.

For full docs, see the [repository README](https://github.com/Hermsi1337/claudex#readme).

## Commands

- `/claudex <task>` — decompose, delegate code work to Codex, do docs in Claude, then review.
  - `--model <name>` — override Codex' model for this call. Without it, Codex' own default is used.
  - `--effort <low|medium|high|xhigh>` — force a Codex reasoning level for this call (bypasses auto-classification).
- `/claudex:setup` — verify the local `codex` CLI is installed and authenticated.

After the first `/claudex` in a conversation, task-shaped follow-ups are routed through the same workflow automatically (sticky mode). Questions about the diff or codebase still get answered directly without delegating.

Without `--effort`, complex subtasks are auto-escalated to `-c model_reasoning_effort=high` (skipped if your `~/.codex/config.toml` default is already `high` or `xhigh`); standard subtasks run at your configured default. Claude never auto-lowers reasoning and never auto-escalates to `xhigh` — `low` and `xhigh` only ever happen when you ask for them via `--effort`.

Every Codex call is also launched with `-c model_verbosity=low` and a prompt rule that tells Codex to skip preamble/recap and produce a ≤5-bullet final summary. Claude reviews via `git diff`, so verbose Codex prose would be wasted tokens. No flag to opt out yet.

## Requirements

- OpenAI Codex CLI: `npm install -g @openai/codex`, then `codex login`.
- Claude Code on macOS or Linux (Windows via WSL).

The plugin uses your local Codex auth via subprocess — no API keys configured in the plugin.
