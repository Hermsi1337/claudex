# claudex — Agent context

This is the **claudex** Claude Code plugin. It turns Claude into an orchestrator that delegates implementation work to OpenAI Codex (via the local `codex` CLI), handles documentation/conceptual work itself, and reviews Codex' diff at the end.

End-user documentation lives in [README.md](README.md). **This** file is for any AI agent working **on** this repo (Claude Code, Codex, Cursor, etc.). `AGENTS.md` is a symlink to this file so every agent reads the same rules.

## Repository layout

- `.claude-plugin/marketplace.json` — marketplace manifest
- `plugins/claudex/.claude-plugin/plugin.json` — plugin manifest
- `plugins/claudex/commands/claudex.md` — `/claudex` orchestrator command
- `plugins/claudex/commands/setup.md` — `/claudex:setup` install/auth checker
- `plugins/claudex/README.md` — short plugin-level README
- `tests/scenarios/*.md` — manual smoke-test specs for `/claudex`
- `tests/sandboxes/` — gitignored work dirs used when running smoke tests
- `README.md` — main user-facing docs
- `LICENSE` — MIT

The `commands/*.md` files are **instructions for Claude**, not scripts. They use `$ARGUMENTS` for user-input substitution and `${CLAUDE_PLUGIN_ROOT}` for plugin-relative paths. Frontmatter follows the official Claude Code plugin spec: `description`, `argument-hint`, `allowed-tools`.

## Required behaviours (always)

### 1. Repo language is English

Every committed artefact in this repo is in English. No exceptions. That includes:

- code, identifiers, inline comments
- docs (`README.md`, `CLAUDE.md`, plugin `README.md`, command `.md` files)
- commit messages, PR titles, PR bodies
- issue/PR replies that get committed (templates etc.)
- error messages, log strings, user-facing text

If the human is chatting in another language, reply to them in their language — but **anything you write into a file or send through `git`/`gh` stays English**.

### 2. Doc & PR-body sync check before every commit

Before running `git commit`, do this — every time, no shortcuts:

1. Look at what you're about to commit: `git diff --staged` (or `git diff HEAD` if nothing is staged yet).
2. Decide whether the change touches anything **user-visible**: install steps, command names, flags, frontmatter, workflow phases, requirements, known limitations, defaults.
3. If yes, verify these are still accurate and update them in the **same commit**:
   - `README.md` (top-level)
   - `plugins/claudex/README.md`
   - `plugins/claudex/commands/claudex.md` — especially the frontmatter (`argument-hint`, `allowed-tools`), the `--model` handling, and the phase descriptions
   - `plugins/claudex/commands/setup.md` — install/auth steps
   - `CLAUDE.md` (this file — repo-layout section in particular)
4. If a PR exists for the current branch, check the PR body too:
   ```bash
   gh pr view --json body -q .body
   ```
   If the body now misrepresents the change set, update it with `gh pr edit --body-file <file>` before finishing.

Quick way to surface drift candidates:

```bash
git diff --staged --name-only | xargs -I{} grep -l -E '(claudex|codex|--model|--sandbox|--cd|setup|/plugin|sticky)' {} 2>/dev/null
```

Triggers that almost always require a doc update: changing a flag, changing a phase description, changing an install step, changing a known-limitations bullet, renaming a command file.

Doc drift is cheap to prevent in the same commit and expensive to clean up later. Don't defer it.

## Plugin design notes worth keeping in mind

- The plugin shells out to `codex exec --sandbox workspace-write` directly (Codex CLI ≥ 0.128). **No Node companion script** — intentionally simpler than `openai/codex-plugin-cc` (which uses `codex app-server` + a `codex-companion.mjs` wrapper). Don't add one without the user asking.
- The legacy `--full-auto` preset and `--ask-for-approval on-failure` are gone in current Codex versions; do not reintroduce them. `codex exec` is non-interactive and has no approval-prompt flag at all.
- When the user pins the work to a subdirectory, the orchestrator passes `--cd <dir>` to Codex so `workspace-write` can't reach files outside it. This is what makes `tests/sandboxes/<scenario>/` actually safe.
- Sticky mode: `claudex.md` instructs Claude to keep applying the same orchestration to task-shaped follow-ups in the same conversation, even without the user re-typing `/claudex`. Questions and design Q&A do not trigger it; an explicit "no more delegating" turns it off for that turn.
- Codex starts each invocation cold. Anything it needs goes into the prompt — context, file scope, conventions, acceptance criteria.
- Prompts are **always** delivered via stdin from a file the orchestrator writes with the Write tool to `<resolved-call-dir>/<slug>-<id>.md`, then `codex exec [flags] - < <file>`. Never inline-quote a prompt as a positional `codex exec` argument and never use a heredoc — both go through the bash parser and have repeatedly produced shell-escaping errors on prompts containing backticks, `$`, nested quotes, or heredoc-delimiter-like substrings. The Write-tool path bypasses bash entirely. Prompt files are intentionally not cleaned up; the prompt root lives under the OS temp dir and the files are useful for debugging botched runs.
- The prompt directory is **per-call**, not shared across calls or sessions. Each `/claudex` call resolves its own subdirectory `<prompt-root>/<call_id>/` in Phase 3 — `call_id` is `<UTC-ish timestamp>-<6 hex chars>` (e.g. `20260509-171152-89c895`). Sticky-mode follow-ups are separate calls and each get their own `call_id`. This isolates concurrent sessions cleanly (no filename races on parallel `/claudex` runs) and makes forensics trivial: every prompt under `<call_id>/` belongs to the same call. The resolution Bash command is `mkdir -p /tmp/claudex-prompts && call_id=... && call_dir=/tmp/claudex-prompts/$call_id && mkdir -p "$call_dir" && (cd "$call_dir" && pwd -W 2>/dev/null || pwd)` — the trailing `pwd -W` is what makes Windows work. On Mac/Linux it falls back to `pwd`; on Git Bash it returns the Windows-side absolute path (typically under `C:/Users/<user>/AppData/Local/Temp/claudex-prompts/<call_id>`) so Claude Code's Write tool and the Bash `<` redirect agree on file location. Without `pwd -W` on Windows, Write resolves `/tmp/...` against the current drive (lands at `D:/tmp/...`) while Bash's `/tmp` is `$TMPDIR` — they're different real directories and the redirect fails with `bash: /tmp/.../foo.md: No such file or directory`. When you change the prompt-delivery flow, keep both the per-call subdir and the `pwd -W` resolution intact, and use the same resolved path for both the Write tool and the redirect.
- Sandbox behaviour is **platform-split**, and the split is load-bearing — don't merge the two paths back together:
  - **macOS/Linux:** Codex' `--sandbox workspace-write` only actually permits writes when the project is in `~/.codex/config.toml` under `[projects."<path>"] trust_level = "trusted"`. Without trust, every patch is rejected with `patch rejected: writing is blocked by read-only sandbox`. The runtime `-c projects.X.trust_level="trusted"` override has been observed not to work reliably, so don't rely on it. Two integrated mitigations: (1) `/claudex:setup` Step 4 detects untrusted projects and offers to append the entry (with explicit user consent via `AskUserQuestion`); (2) Phase 3 has a runtime fallback that re-runs the same `codex exec` call once with `--dangerously-bypass-approvals-and-sandbox` if and only if the output contains the exact substring `patch rejected: writing is blocked by read-only sandbox`. The fallback must be reported prominently in Phase 5 so the user knows the call ran with sandboxing disabled.
  - **Native Windows:** the sandbox is unavailable, full stop. Confirmed during smoke tests on Codex CLI 0.128: `--sandbox workspace-write` always degrades to read-only (session header shows `sandbox: read-only`) and rejects every patch **regardless of trust-list entries** — repo-root entry, exact `--cd` subdir entry, lowercased-backslash and exact-case TOML forms were all tested and all ineffective. Consequently: Phase 3 detects native Windows (drive-letter result from the prompt-dir resolution) and adds `--dangerously-bypass-approvals-and-sandbox` to every write-capable call from the start (a sandboxed first attempt is a guaranteed wasted round-trip), Phase 5 carries a notice that this is the expected steady state, and `/claudex:setup` Step 4 skips the trust flow on Windows (a trust entry would be dead config). If a future Codex release fixes Windows sandboxing, unwind the special-casing in `claudex.md`, `setup.md`, both READMEs, and this bullet together.
  - The bypass flag is documented on `codex exec --help` for Codex CLI ≥ 0.128 — keep the existing rule that the legacy `--full-auto` preset is gone, but don't claim sandbox+approval flags alone are sufficient.
- Parallelism is decided in natural language inside `claudex.md`, not enforced by code. Rule: file-disjoint Codex subtasks may run in parallel as `Bash` calls with `run_in_background: true`; anything sharing a file runs sequentially. When in doubt, sequential.
- Default Codex model is whatever `codex` itself defaults to (so users automatically get its latest). `/claudex --model <name>` overrides for one call.
- Reasoning effort is decided in this priority order: (1) `--effort <low|medium|high|xhigh>` if the user passed it — wins for every subtask in that call, including `low`. (2) Otherwise Phase 1 tags each Codex subtask `complex` or `standard`; Phase 3 adds `-c model_reasoning_effort=high` only for `complex` ones, and only if a Phase 2 peek at `${CODEX_HOME:-$HOME/.codex}/config.toml` shows the user's default isn't already at or above `high` (i.e. `high` or `xhigh`). (3) `standard` subtasks get no override and run at the user's configured default. The orchestrator never auto-sets `low`, and never auto-escalates to `xhigh` either — auto-escalation tops out at `high`; `xhigh` is reserved for explicit `--effort xhigh`. Effort ordering: `low < medium < high < xhigh`.
- Codex output verbosity is forced to `low` (`-c model_verbosity=low`) on every `codex exec` call, with no user-facing override. Rationale: claudex reviews Codex' work via `git diff`, not via Codex' assistant prose, so verbose narration is pure token waste. The orchestrator pairs this with an explicit "output discipline" rule in the prompt template (Phase 3 must-include #6: no preamble, ≤5-bullet final summary, errors quoted verbatim) so the final summary stays useful even at low verbosity. If a real use case for verbose Codex output ever appears, add a `--verbose` flag — do **not** make this default-off, and do not extend the per-subtask classification to flip verbosity. One always-on knob is the entire point.
- The Phase 3 hand-off rule (must-include #5) additionally forbids Codex from running steps the orchestrator handles itself: `git status`/`diff`/`log`/`show`, formatters (`gofmt`, `goimports`, `prettier`, `black`, `rustfmt`, etc.), and repeated test runs after applying fixes. Codex is also told to open additional files only when a build/test failure names one — embedded excerpts are the source of truth. Same rationale as low verbosity: those steps' output gets pulled back into Codex' own context for no review benefit (Phase 4 already runs them on the orchestrator side). Real-world impact, observed on a single substantive subtask: ~70k → ~35–40k tokens, dominated by killing repeated `git diff` dumps and the duplicate test run Codex used to do "to be sure". When you change this forbidden-list, also check that it still matches what `README.md` ("Execution discipline") and `plugins/claudex/README.md` describe.
- Trivial code subtasks (single-edit, fully-specified, ≤2 files, no test/build loop, ≤2 files of context to read) stay in Claude — Phase 1 carves them out as an exception to "code → Codex". Same intent as the existing doc/code split: only round-trip to Codex when Codex actually adds value over a single Edit call. The trivial filter is intentionally narrow and the tie-break rule is "when in doubt → Codex"; the user invoked `/claudex` because they wanted Codex doing the code work. Don't broaden the trivial criteria without an explicit ask — the safer failure mode is one wasted Codex call, not a Claude edit that misses something.
- `/claudex:setup` deliberately does **not** run `codex login` — that's an interactive browser flow that doesn't work inside Claude Code.

## Smoke tests (`tests/`)

Manual end-to-end scenarios for `/claudex` live under `tests/scenarios/`. Each `.md` file is a self-contained recipe (Goal / Setup / Invocation / Expected / Verify / Cleanup). There's no runner — humans (or another agent) read a scenario and execute it against a real Codex CLI.

- `tests/sandboxes/<scenario>/` is the per-test work dir. Its contents are gitignored (the dir is tracked only by its `.gitignore`), so test runs don't dirty the working tree.
- **Codex sees the entire repo as its workspace** when `/claudex` runs from inside this repo. The scenarios always include "do not modify anything outside `tests/sandboxes/<scenario>/`" in the prompt, but that's a soft constraint. Always verify with `git status --porcelain -- ':(exclude)tests/sandboxes/'` after each run; reset with `git checkout --` anything that escaped.
- When you change `plugins/claudex/commands/claudex.md` in a way that affects decomposition, parallelism, or review behaviour, run scenarios 01–03 at minimum. They are quick.
- When you add a new behaviour (a new flag, a new phase), add a matching `tests/scenarios/NN-*.md` in the same commit. Same-commit doc rule applies.

## Out of scope for now

Tracked in [README.md](README.md) under "Not in this version". Don't expand scope without the user explicitly asking — particularly: no iteration loop, no session resume, no background-mode commands.
