# claudex (plugin)

The `claudex` plugin makes Claude Code an orchestrator for OpenAI Codex.

For full docs, see the [repository README](https://github.com/Hermsi1337/claudex#readme).

## Commands

- `/claudex <task>` — decompose, delegate code work to Codex, do docs in Claude, then review.
  - `--model <name>` — override Codex' model for this call. Without it, Codex' own default is used.
- `/claudex:setup` — verify the local `codex` CLI is installed and authenticated.

After the first `/claudex` in a conversation, task-shaped follow-ups are routed through the same workflow automatically (sticky mode). Questions about the diff or codebase still get answered directly without delegating.

## Requirements

- OpenAI Codex CLI: `npm install -g @openai/codex`, then `codex login`.
- Claude Code on macOS or Linux (Windows via WSL).

The plugin uses your local Codex auth via subprocess — no API keys configured in the plugin.
