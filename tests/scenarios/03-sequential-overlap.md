# 03 — sequential, same-file overlap

## Goal

Verify that two Codex subtasks that both touch the same file run **sequentially**, not in parallel. This is the failure mode that causes silent merge conflicts and is the single most important parallelism rule in `claudex.md`.

## Setup

```bash
rm -rf tests/sandboxes/03-sequential
mkdir -p tests/sandboxes/03-sequential
cat > tests/sandboxes/03-sequential/calc.py <<'PY'
def add(a, b):
    return a + b
PY
```

## Invocation

```
/claudex In tests/sandboxes/03-sequential/calc.py: (1) add a multiply(a, b) function that returns a * b; (2) add a divide(a, b) function that returns a / b and raises a clear ZeroDivisionError with a helpful message when b == 0. Do not modify anything outside tests/sandboxes/03-sequential/.
```

## Expected behaviour

- **Phase 1:** split into 2 Codex subtasks. **Both touch `calc.py`** → file overlap → **must run sequentially**. Claude should explicitly call this out in its plan.
- **Phase 3:** **two Codex jobs, strictly one after the other.** Each gets its own Write-tool prompt file under the same per-call subdirectory (resolved once at the start of this `/claudex` call), a background `codex exec [flags] - < <name>.md > <name>.out 2>&1; echo $? > <name>.exit` launch, and a sentinel watcher. Background launching is expected — it's how every Codex job runs — but the **second job's launch must happen only after the first job's `.exit` sentinel confirmed success**. **No** inline-quoted prompts and **no** heredocs anywhere.
- **Phase 4:** review reads the final `calc.py`, confirms both new functions are present, the original `add` is intact, and divide raises `ZeroDivisionError` for `b == 0`.
- **All platforms:** `--dangerously-bypass-approvals-and-sandbox` must **not** appear unless the runtime fallback of scenario 11 fired — Codex ≥ 0.145 sandboxes native Windows too (see scenario 14).

## Failure mode to watch for

If Claude launches both subtasks in parallel anyway, you'll see both `codex exec` launches in the same tool batch (or the second launched before the first's `.exit` sentinel existed). The diff will likely show one function lost (last write wins) or merge-conflict markers. That's a **bug in the orchestrator's overlap analysis** — log it.

## Verify

```bash
cat tests/sandboxes/03-sequential/calc.py

python -c "
import sys; sys.path.insert(0, 'tests/sandboxes/03-sequential')
from calc import add, multiply, divide
assert add(2, 3) == 5,        'add broken'
assert multiply(2, 3) == 6,   'multiply broken'
assert divide(6, 2) == 3.0,   'divide broken'
try:
    divide(1, 0)
except ZeroDivisionError as e:
    print('divide-by-zero raised correctly:', e)
else:
    raise SystemExit('divide(1, 0) should raise ZeroDivisionError')
"

# nothing outside the sandbox
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

## Cleanup

```bash
rm -rf tests/sandboxes/03-sequential
```
