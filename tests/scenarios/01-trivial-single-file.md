# 01 — trivial single-file

## Goal

Verify that for a tiny one-file change, the orchestrator does **not** over-split the task: one Codex invocation, no parallelism, brief review, clean report.

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

- **Phase 1 (decompose):** no split — orchestrator recognises this as trivial.
- **Phase 2 (context):** runs `git status` / brief glob; no need to read existing files because the sandbox is empty.
- **Phase 3 (execute):** exactly one Bash call of the form `codex exec --sandbox workspace-write --cd tests/sandboxes/01-trivial "..."`. No `--model` flag (we didn't pass one). No `run_in_background: true` (nothing to parallelise with).
- **Phase 4 (review):** reads `hello.py`. No comments expected (default policy). Confirms the `__main__` block is present and calls `greet`.
- **Phase 5 (report):** 1 delegated to Codex, 0 done by Claude. Recommendation should be `commit` (or whatever phrasing — point is, no issues).

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

## Cleanup

```bash
rm -rf tests/sandboxes/01-trivial
```
