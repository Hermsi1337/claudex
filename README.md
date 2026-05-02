# claudex

Claude Code plugin that turns Claude into an **orchestrator** and delegates concrete implementation work to **OpenAI Codex** (running locally via the `codex` CLI).

The idea: Claude is great at conceptual work, decomposition and review. Codex is fast at well-scoped implementation tasks. With `claudex`, you stay in Claude Code, hand it a task, and Claude:

1. Splits the task into subtasks.
2. Delegates code-changing subtasks to Codex (in parallel where it's safe).
3. Handles documentation/conceptual subtasks itself.
4. Reviews Codex' diff.
5. Reports back with what changed and what to do next.

Inspired by [`openai/codex-plugin-cc`](https://github.com/openai/codex-plugin-cc), but with the inverse flow: that plugin uses Codex primarily for code review and bug rescue. `claudex` is **Claude → Codex**, not the other way around.

## Requirements

- [Claude Code](https://claude.com/claude-code) (CLI, IDE extension, or desktop)
- [OpenAI Codex CLI](https://github.com/openai/codex) **≥ 0.128** installed and authenticated locally
  - `npm install -g @openai/codex`
  - `codex login`
  - The plugin uses `codex exec --sandbox workspace-write [--cd <dir>] [--model <name>]`. Older Codex versions had a `--full-auto` preset that this plugin no longer relies on.
- macOS or Linux (Windows via WSL or directly with Git Bash + a working `codex`)

The plugin shells out to `codex exec` and uses your existing Codex auth — no separate API key configuration.

## Install

In Claude Code:

```
/plugin marketplace add Hermsi1337/claudex
/plugin install claudex@claudex
```

Then verify your Codex CLI is ready:

```
/claudex:setup
```

## Usage

### Simple case

```
/claudex add a healthcheck endpoint at /api/health
```

Claude reads the relevant files, sends one self-contained prompt to Codex, then reviews the diff.

### Complex case (parallel + sequential)

```
/claudex add user signup with email verification, write API docs, and add integration tests
```

Claude splits this into subtasks, identifies file overlaps, and runs them with the right concurrency:

- Signup implementation (Codex) ‖ API docs (Claude itself) — parallel, file-disjoint
- Integration tests (Codex) — sequential, after the signup code lands

Then Claude reviews everything Codex produced and reports back.

### Override the model

```
/claudex --model gpt-5.6-codex add caching to the user lookup
```

Without `--model`, Codex' own default is used (so you always get its latest model).

### Sticky follow-ups

After your first `/claudex` in a conversation, follow-up task-shaped requests in the same chat are routed through the same orchestrator automatically — no need to retype `/claudex`:

```
/claudex add a healthcheck endpoint at /api/health
… (Claude delegates, reviews, reports)

now also add a /api/version endpoint that returns the package version
… (Claude does the same workflow without you typing /claudex again)
```

Questions about the diff or the codebase don't trigger a Codex run; only sentence-shaped task requests do. To turn it off explicitly: "just answer me directly" or "no more delegating".

## What gets delegated to Codex

| Subtask kind | Handled by |
| --- | --- |
| Implementing a feature | Codex |
| Refactoring | Codex |
| Writing tests | Codex |
| Mechanical edits across many files | Codex |
| READMEs, ADRs, design notes | Claude |
| Commit messages, PR descriptions | Claude |
| Architectural decisions | Claude |

## Inline-comment policy

Codex follows whatever it finds in your `CLAUDE.md` / `AGENTS.md` / `.codex/config.toml`. If nothing is specified, the default is **self-explanatory code, no inline comments** — Claude tells Codex this explicitly when it delegates.

## Parallelism rules

- File-disjoint Codex subtasks run **in parallel** (Claude launches them as background bash jobs while it works on docs).
- Subtasks that share any file run **sequentially**.
- When in doubt, sequential.

If Claude misjudges disjointness and two parallel jobs touch the same file, the review step flags it and recommends a revert.

## Commands

- `/claudex <task>` — orchestrate + delegate + review.
  - `--model <name>` — override Codex' default model for this call.
- `/claudex:setup` — verify Codex CLI is installed and authenticated.

## Not in this version

These are tracked as future work:

- [ ] Iteration loop (Claude reviews → re-delegates fixes to Codex automatically)
- [ ] Resume of Codex sessions
- [ ] Configurable reasoning-effort flag (e.g. `--effort high`)
- [ ] Background mode with `/claudex:status` and `/claudex:result`

## License

MIT — see [LICENSE](LICENSE).
