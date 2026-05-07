# 01 — trivial single-file (kept in Claude)

## Goal

Verify the Phase 1 trivial-code carve-out fires for a tiny, fully-specified one-file change: Claude applies the change itself with `Write` and **does not invoke Codex at all**. No `codex exec` call, no `/tmp/claudex-prompts/` file, no parallelism. The whole orchestration collapses to one Write tool call plus the report.

## Setup

```bash
rm -rf tests/sandboxes/01-trivial
mkdir -p tests/sandboxes/01-trivial
```

## Invocation

```
/claudex Inside tests/sandboxes/01-trivial/, create a file hello.py with a function greet(name: str) -> None that prints "hello, <name>!" to stdout, plus an `if __name__ == "__main__":` block that calls greet("world"). Do not modify anything outside tests/sandboxes/01-trivial/.
```

## Expected behaviour

- **Phase 1 (decompose):** the orchestrator classifies this as trivial-code. All four criteria hold:
  1. The change is fully specified (filename, function signature, exact `__main__` block).
  2. Single file.
  3. No test or build loop required to produce the change.
  4. Sandbox is empty, so no other files to read.

  → keep the work in Claude.
- **Phase 2 (context):** `git status` / brief glob; sandbox is empty so there are no files to read.
- **Phase 3 (execute):** **no `codex exec` call at all.** No prompt file written under `/tmp/claudex-prompts/` for this run. The orchestrator uses the `Write` tool directly to create `tests/sandboxes/01-trivial/hello.py`.
- **Phase 4 (review):** Claude does **not** review its own Write output (per the Phase 4 rule "do not review your own work"). It may spot-check by reading the file back, but no formal review section is required.
- **Phase 5 (report):** `0 delegated to Codex`, `1 done by Claude` (the file write). Recommendation: `commit`.

## Verify

```bash
ls tests/sandboxes/01-trivial/
cat tests/sandboxes/01-trivial/hello.py
python tests/sandboxes/01-trivial/hello.py
# expected stdout: hello, world!

# nothing outside the sandbox should have changed
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty output
```

## Failure modes to watch for

- A `codex exec` call happens at all → the trivial-code carve-out is not firing. Check Phase 1's classification logic in [claudex.md](../../plugins/claudex/commands/claudex.md).
- A prompt file appears under `/tmp/claudex-prompts/` for this run → same regression: the orchestrator went to Codex when it shouldn't have.
- The orchestrator over-classifies trivial as standard-code "to be safe" — the whole point of the carve-out is to skip Codex when a single Edit/Write would suffice. If every trivial task still round-trips through Codex, the carve-out is dead. Confirm the report says `0 delegated to Codex`.
- The orchestrator splits this into two subtasks (e.g. "create file" + "add main block") → over-decomposition; classify as trivial and do it in one Write.

## Cleanup

```bash
rm -rf tests/sandboxes/01-trivial
```
