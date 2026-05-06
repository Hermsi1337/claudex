# 07 — `--effort` flag override

## Goal

Verify that `/claudex --effort <level> <task>` parses the flag, strips it from the task description, and forwards `-c model_reasoning_effort=<level>` to **every** `codex exec` call in that invocation — including levels that auto-classification would never set on its own (`low`, `xhigh`).

## Setup

```bash
rm -rf tests/sandboxes/07-effort
mkdir -p tests/sandboxes/07-effort
```

## Invocation A — `--effort low` on a complex-looking task

The task is genuinely complex (would normally auto-escalate to `high`). With `--effort low`, the user override must win.

```
/claudex --effort low In tests/sandboxes/07-effort/, design and implement an in-memory LRU cache with TTL eviction in cache.py and unit tests in test_cache.py covering size and TTL eviction. Do not modify anything outside tests/sandboxes/07-effort/.
```

### Expected behaviour

- **Argument parsing:** Claude removes `--effort low` from the natural-language task. The Codex prompt should not contain the literal string `--effort`.
- **Phase 3:** every `codex exec` call carries `-c model_reasoning_effort=low`. Roughly:
  ```bash
  codex exec --sandbox workspace-write -c model_reasoning_effort=low --cd tests/sandboxes/07-effort "..."
  ```
- The auto-classification that would have escalated this task to `high` is overridden — the `low` from `--effort` wins.

## Invocation B — `--effort xhigh` on a trivial task

The task is trivial (would normally get no override). With `--effort xhigh`, the user override must still apply.

Run in the same conversation (sticky mode) to also verify the override travels through sticky calls:

```
now with --effort xhigh, in tests/sandboxes/07-effort/, rename cache.py's TTLCache class to TimedCache and update test_cache.py accordingly. Do not modify anything outside tests/sandboxes/07-effort/.
```

### Expected behaviour

- **Argument parsing:** `--effort xhigh` is stripped from the task description.
- **Phase 3:** Bash call carries `-c model_reasoning_effort=xhigh`:
  ```bash
  codex exec --sandbox workspace-write -c model_reasoning_effort=xhigh --cd tests/sandboxes/07-effort "..."
  ```
- The fact that the rename is a `standard` subtask is irrelevant — the user override wins.

## Invocation C — invalid level

```
/claudex --effort ultra In tests/sandboxes/07-effort/, write a noop.py file with a single function noop() that returns None.
```

### Expected behaviour

- The orchestrator rejects `ultra` as an invalid effort level (allowed: `low`, `medium`, `high`, `xhigh`) and asks the user to fix it. **No** `codex exec` call is made.

## Verify

Inspect the transcript for invocations A and B and confirm:

- The exact `-c model_reasoning_effort=<level>` segment is present on every Codex Bash call.
- For invocation C, no `codex exec` was launched.

```bash
ls tests/sandboxes/07-effort/
# after A: cache.py, test_cache.py
# after B: same files, with TTLCache → TimedCache renamed
# after C: nothing new (the orchestrator should have stopped before launching Codex)

git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

If your client hides the underlying Bash command, fall back to the trick from scenario 04: re-run with `--effort definitely-not-a-level` and expect the orchestrator to refuse before invoking Codex — that's a positive signal that flag parsing is happening.

## Cleanup

```bash
rm -rf tests/sandboxes/07-effort
```
