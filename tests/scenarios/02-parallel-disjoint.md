# 02 — parallel, file-disjoint (code + doc)

## Goal

Verify that a code subtask and a doc subtask on disjoint files run in parallel: Claude launches the Codex job in the background and writes the README itself in the meantime.

## Setup

```bash
rm -rf tests/sandboxes/02-parallel
mkdir -p tests/sandboxes/02-parallel
```

## Invocation

```
/claudex In tests/sandboxes/02-parallel/: (1) implement an in-memory key-value store in store.py — a class KVStore exposing put(key, value), get(key, default=None), delete(key), and clear(); (2) write a README.md in the same folder explaining how to use the class with a short usage example. Do not modify anything outside tests/sandboxes/02-parallel/.
```

## Expected behaviour

- **Phase 1:** split into 2 subtasks.
  - `store.py` → **Codex** (code)
  - `README.md` → **Claude** itself (doc)
  - Files are disjoint → eligible for parallel execution.
- **Phase 3:** Claude first writes the Codex prompt to a `/tmp/claudex-prompts/<call_id>/<slug>-<id>.md` file via the Write tool (the `<call_id>` subdirectory is resolved once at the start of this `/claudex` call), then launches `codex exec [flags] - < <name>.md > <name>.out 2>&1; echo $? > <name>.exit` with `run_in_background: true` plus a sentinel watcher, and **then** immediately uses `Write` / `Edit` on `README.md`. The Codex Bash call and the README work should be in flight at roughly the same time. Once the watcher (or any wake-up) fires, Claude reads the exit code from `<name>.exit` and the summary from `<name>.out` — it does not idle waiting for the Codex job's own background-task notification.
- **Phase 4:** review covers `store.py` only (Claude does not review its own README).
- **Phase 5:** 1 delegated to Codex, 1 done by Claude.
- **All platforms:** `--dangerously-bypass-approvals-and-sandbox` must **not** appear unless the runtime fallback of scenario 11 fired — Codex ≥ 0.145 sandboxes native Windows too (see scenario 14).

## Verify

```bash
ls tests/sandboxes/02-parallel/
test -f tests/sandboxes/02-parallel/store.py && echo "store.py: OK"
test -f tests/sandboxes/02-parallel/README.md && echo "README.md: OK"

# functional smoke
python -c "
import sys; sys.path.insert(0, 'tests/sandboxes/02-parallel')
from store import KVStore
s = KVStore()
s.put('a', 1); assert s.get('a') == 1
s.delete('a'); assert s.get('a') is None
s.put('b', 2); s.clear(); assert s.get('b') is None
print('store.py: behaves as expected')
"

# README should mention KVStore and at least one method by name
grep -E 'KVStore|put|get|delete' tests/sandboxes/02-parallel/README.md > /dev/null && echo "README.md: references API"

# nothing outside the sandbox
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

## Cleanup

```bash
rm -rf tests/sandboxes/02-parallel
```
