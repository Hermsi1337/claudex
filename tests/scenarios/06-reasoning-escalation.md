# 06 — Reasoning-effort auto-escalation

## Goal

Verify that the orchestrator picks `-c model_reasoning_effort=high` for genuinely complex Codex subtasks and omits the override entirely for standard ones. The output of Codex itself isn't what we're testing — only that Phase 1 classification ends up as the right `-c …` flag (or absence thereof) on the actual `codex exec` Bash call.

## Setup

```bash
rm -rf tests/sandboxes/06-reasoning
mkdir -p tests/sandboxes/06-reasoning
```

## Invocation A — standard subtask (no override expected)

```
/claudex In tests/sandboxes/06-reasoning/, create greet.py with a function greet(name) that returns "hello, " + name. Do not modify anything outside tests/sandboxes/06-reasoning/.
```

### Expected behaviour

- Phase 1: subtask classified as `standard`.
- Phase 3 Bash call **must not** contain `-c model_reasoning_effort=`. It looks roughly like:
  ```bash
  codex exec --sandbox workspace-write --cd tests/sandboxes/06-reasoning "..."
  ```

## Invocation B — complex subtask (high expected)

Run this in the same conversation (sticky mode), so we also confirm the heuristic fires on follow-ups:

```
now in tests/sandboxes/06-reasoning/, design and implement a small in-memory LRU cache with TTL eviction in cache.py — pick the data structures yourself, handle concurrent access, and add unit tests in test_cache.py covering eviction by size, eviction by TTL, and the interaction between the two. Do not modify anything outside tests/sandboxes/06-reasoning/.
```

### Expected behaviour

- Phase 1: subtask(s) classified as `complex` (algorithm/data-structure design + multi-file work + non-trivial test design).
- Phase 3 Bash call(s) **must** contain `-c model_reasoning_effort=high`. Roughly:
  ```bash
  codex exec --sandbox workspace-write -c model_reasoning_effort=high --cd tests/sandboxes/06-reasoning "..."
  ```
  Order of flags is irrelevant; the `-c model_reasoning_effort=high` segment must be present.

## Verify

Inspect the transcript for both invocations and confirm the flag presence/absence matches what's described above. The actual files Codex produced are secondary, but a basic sanity check:

```bash
ls tests/sandboxes/06-reasoning/
# expect at least: greet.py, cache.py, test_cache.py

# nothing outside the sandbox
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

If your client hides the underlying Bash command (e.g. some Claude Desktop builds), look for the orchestrator's Phase 1 todo list / classification output instead — it should explicitly say `standard` for invocation A and `complex` for invocation B. If that's also not visible, this scenario can't be verified in that client.

## Negative check (optional)

Make sure the orchestrator never silently sets `low`. Run:

```
/claudex In tests/sandboxes/06-reasoning/, rename greet to say_hello in greet.py.
```

The Bash call must again contain **no** `-c model_reasoning_effort=` flag — not `low`, not anything. Auto-lowering is explicitly forbidden.

## Cleanup

```bash
rm -rf tests/sandboxes/06-reasoning
```
