# 06 — Reasoning-effort auto-escalation

## Goal

Verify the orchestrator's auto-classification → `-c model_reasoning_effort=…` mapping when no `--effort` flag is given:

- `standard` subtasks → no `-c model_reasoning_effort=` flag.
- `complex` subtasks → `-c model_reasoning_effort=high`, **unless** the user's `~/.codex/config.toml` already configures `high` or `xhigh` (Phase 2 peek), in which case the flag is omitted.
- The orchestrator must never auto-lower (`low`) or auto-jump to `xhigh`.

The output of Codex itself isn't under test here — only the flag composition on the actual `codex exec` Bash call.

## Setup

```bash
rm -rf tests/sandboxes/06-reasoning
mkdir -p tests/sandboxes/06-reasoning

# Capture your current Codex default for context — needed to interpret invocation B.
grep -E '^[[:space:]]*model_reasoning_effort[[:space:]]*=' \
  "${CODEX_HOME:-$HOME/.codex}/config.toml" 2>/dev/null | head -1
# Expected output examples:
#   model_reasoning_effort = "medium"   (or "low", or unset)  → invocation B should escalate
#   model_reasoning_effort = "high"     or  "xhigh"           → invocation B should skip the override
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

## Invocation B — complex subtask (high expected, unless your default is already ≥ high)

Run this in the same conversation (sticky mode), so we also confirm the heuristic fires on follow-ups:

```
now in tests/sandboxes/06-reasoning/, design and implement a small in-memory LRU cache with TTL eviction in cache.py — pick the data structures yourself, handle concurrent access, and add unit tests in test_cache.py covering eviction by size, eviction by TTL, and the interaction between the two. Do not modify anything outside tests/sandboxes/06-reasoning/.
```

### Expected behaviour

- Phase 1: subtask(s) classified as `complex`.
- Phase 2: orchestrator peeks at `~/.codex/config.toml`.
- Phase 3 — depends on what the peek found:
  - **Default is `low`, `medium`, missing, or unreadable** → Bash call **must** contain `-c model_reasoning_effort=high`:
    ```bash
    codex exec --sandbox workspace-write -c model_reasoning_effort=high --cd tests/sandboxes/06-reasoning "..."
    ```
  - **Default is already `high` or `xhigh`** → Bash call **must not** contain `-c model_reasoning_effort=` at all. Adding `high` would either be a no-op or, against `xhigh`, would actually lower reasoning.

  Order of flags is irrelevant; what matters is presence vs. absence.

## Verify

Inspect the transcript for both invocations and confirm the flag presence/absence matches the rule that applies to your config. Sanity check the produced files:

```bash
ls tests/sandboxes/06-reasoning/
# expect at least: greet.py, cache.py, test_cache.py

# nothing outside the sandbox
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

If your client hides the underlying Bash command (e.g. some Claude Desktop builds), look for the orchestrator's Phase 1 todo list / classification output and the Phase 2 peek summary instead. If those are also not visible, this scenario can't be verified in that client.

## Negative check (optional but recommended)

Make sure the orchestrator never silently sets `low`. Run:

```
/claudex In tests/sandboxes/06-reasoning/, rename greet to say_hello in greet.py.
```

The Bash call must again contain **no** `-c model_reasoning_effort=` flag — not `low`, not anything. Auto-lowering is explicitly forbidden, regardless of how trivial the subtask is.

## Optional: simulate a `high` default to test the skip path

If your real config is below `high` and you want to exercise the skip path without editing your config permanently:

```bash
# Run only this invocation with a fake CODEX_HOME that points to a config that's already at xhigh.
# (This affects only this scenario's runs — your real ~/.codex stays untouched.)
mkdir -p /tmp/claudex-fakecodex
cat > /tmp/claudex-fakecodex/config.toml <<'TOML'
model_reasoning_effort = "xhigh"
TOML
CODEX_HOME=/tmp/claudex-fakecodex /claudex In tests/sandboxes/06-reasoning/, design a tiny ring-buffer with overwrite-on-full in ring.py and unit tests in test_ring.py. Do not modify anything outside tests/sandboxes/06-reasoning/.
```

Expected: `complex` subtask, peek finds `xhigh`, **no** `-c model_reasoning_effort=` flag in the Bash call. Cleanup:

```bash
rm -rf /tmp/claudex-fakecodex
```

## Cleanup

```bash
rm -rf tests/sandboxes/06-reasoning
```
