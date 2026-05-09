# Smoke tests

Manual end-to-end scenarios for `/claudex`. Each `scenarios/NN-*.md` file describes one full run: setup, the exact `/claudex` invocation, what the orchestrator should do, and how to verify after.

These are **smoke tests, not a test suite**. There is no runner, no CI hook. You read a scenario, execute it in a real Claude Code session against a real local Codex CLI, and inspect the result yourself.

## Folder layout

```
tests/
├── README.md           # this file
├── scenarios/          # committed scenario specs
│   ├── 01-trivial-single-file.md
│   ├── 02-parallel-disjoint.md
│   ├── 03-sequential-overlap.md
│   ├── 04-model-override.md
│   ├── 05-sticky-followup.md
│   ├── 06-reasoning-escalation.md
│   ├── 07-effort-flag.md
│   └── 08-prompt-via-stdin.md
└── sandboxes/          # gitignored work dirs (only .gitignore is committed)
```

## Safety: Codex sees the whole repo as workspace

When you run `/claudex` from inside this repo, `codex exec --sandbox workspace-write` uses the workspace dir as its writable sandbox. The orchestrator should pass `--cd tests/sandboxes/<scenario>` for these tests, which scopes the sandbox to that subdir; the scenario prompts also tell Codex in plain language to stay inside it. Both belt and braces. If the orchestrator forgets to pass `--cd`, Codex sees the **entire repo** as workspace and could edit files outside the sandbox.

Two cheap guardrails for every test run:

1. Start each test from a clean working tree (`git status` shows nothing).
2. After the run, check that nothing outside the sandbox changed:
   ```bash
   git status --porcelain -- ':(exclude)tests/sandboxes/'
   ```
   That should print nothing. If it doesn't, Codex went out of scope — note it as a finding and reset:
   ```bash
   git checkout -- <out-of-scope-paths>
   ```

If a test was supposed to leave files in `tests/sandboxes/<N>/`, those won't show in `git status` because the sandbox content is gitignored — that's intentional.

## How to run a scenario

1. Pick `scenarios/NN-*.md` and read the whole thing first.
2. Run the **Setup** block from the repo root (creates the sandbox dir, seeds any starter files).
3. From inside the same Claude Code session that has `claudex` enabled, run the **Invocation** line verbatim (substitute placeholders like `<your-codex-model>`).
4. Compare what the orchestrator does against **Expected behaviour**.
5. Run the **Verify** block.
6. Run the **Cleanup** block when done (or leave the sandbox if you want to inspect the diff).

Don't run two scenarios in parallel — the parallelism guarantees being tested are about Codex subtasks within one `/claudex` call, not between separate calls.

## What to look for, generally

In Phase 1 (decompose) of every run, watch the orchestrator's plan:
- Does it correctly identify which subtasks are code vs doc?
- Does it correctly identify file overlaps?
- Does the parallel-vs-sequential decision match the overlap analysis?

In Phase 4 (review), watch what the orchestrator notices:
- Does it actually read every file Codex modified, or just trust the diff stat?
- Does it flag genuine issues (out-of-scope edits, half-finished impl, unwanted comments)?

If the orchestrator silently skips a phase, that's a bug in `plugins/claudex/commands/claudex.md`, not a Codex problem.

In Phase 3, every Codex call should look like `codex exec [flags] - < /tmp/claudex-prompts/<call_id>/<slug>-<id>.md`. If you ever see the prompt inlined as a positional argument or wrapped in a heredoc, that's a regression — the orchestrator must always go through the prompt-file path. Each `/claudex` call resolves its own per-call subdirectory (timestamped + 6-hex `call_id`) so concurrent sessions don't share prompt files.

## Adding a new scenario

Copy an existing `scenarios/NN-*.md`, increment the number, keep the same section structure (Goal / Setup / Invocation / Expected / Verify / Cleanup). Keep prompts short and explicit about file scope.
