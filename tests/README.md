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
│   └── 04-model-override.md
└── sandboxes/          # gitignored work dirs (only .gitignore is committed)
```

## Safety: Codex sees the whole repo as workspace

When you run `/claudex` from inside this repo, `codex exec --full-auto` uses the **entire repo** as its `workspace-write` sandbox. The scenario prompts always tell Codex to stay inside `tests/sandboxes/<scenario>/`, but a misbehaving Codex run **could** edit files outside that path.

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

## Adding a new scenario

Copy an existing `scenarios/NN-*.md`, increment the number, keep the same section structure (Goal / Setup / Invocation / Expected / Verify / Cleanup). Keep prompts short and explicit about file scope.
