# 09 — Low-verbosity default

## Goal

Verify two things that together implement the always-on token-trimming default:

1. **Flag side.** Every `codex exec` invocation the orchestrator issues includes `-c model_verbosity=low`. There is no flag to opt out and no per-subtask exception — the override must appear on every call, including sticky-mode follow-ups, complex subtasks, and the negative-check below.
2. **Prompt side.** The prompt the orchestrator writes under `/tmp/claudex-prompts/<call_id>/` tells Codex explicitly: no preamble, no recap, summarise the result in at most 5 short bullets in the form `path/to/file: one-line change`, errors quoted verbatim, code blocks only when essential.

If either half is missing, this is a regression.

The invocations below intentionally include a test-suite verification step so they fail the Phase 1 trivial-code carve-out's "no test execution required" criterion and reach Codex. A bare "create greet.py with a function" task would otherwise stay in Claude under the trivial-code rule, leaving no `codex exec` call to inspect.

## Setup

```bash
rm -rf tests/sandboxes/09-verbosity
mkdir -p tests/sandboxes/09-verbosity
```

## Invocation A — single Codex subtask (sticky-off)

```
/claudex In tests/sandboxes/09-verbosity/, create a `Greeter` class in greet.py exposing greet(name) returning "hello, " + name and goodbye(name) returning "bye, " + name. Add tests in test_greet.py covering both methods, and verify the test suite passes by running `python -m pytest test_greet.py`. Do not modify anything outside tests/sandboxes/09-verbosity/.
```

### Expected behaviour

- The actual `codex exec` Bash call must contain `-c model_verbosity=low`. Order of flags is irrelevant. A correct call looks roughly like:
  ```bash
  codex exec --sandbox workspace-write --cd tests/sandboxes/09-verbosity -c model_verbosity=low - < /tmp/claudex-prompts/<call_id>/<slug>-<id>.md
  ```
  (Combined with whatever other flags Phase 3 decided to add, e.g. `-c model_reasoning_effort=…` — but `model_verbosity=low` is unconditional.)
- Open the prompt file under `/tmp/claudex-prompts/<call_id>/` that this run produced (the per-call subdirectory created when the call started). It must contain language that maps to the must-include #6 rule: an explicit "no preamble / no recap / no narration" instruction, a "summarise in at most 5 short bullets" instruction, and "quote errors verbatim". Exact wording is up to the orchestrator; the substance must be there.
- Codex' own final assistant message in the transcript should be short: no "Sure, I'd be happy to help" preamble, no recap of the task, a final summary of at most 5 bullets per file changed. Rule of thumb for this single-subtask task: ≤ 15 lines of Codex prose total.

## Invocation B — sticky follow-up

In the same conversation, without retyping `/claudex`:

```
now extend Greeter with a welcome_back(name) method returning "welcome back, " + name. Add a test for it in test_greet.py and re-verify the test suite passes.
```

### Expected behaviour

- Sticky mode triggers (this is a task-shaped follow-up) and Phase 3 fires again.
- The new `codex exec` call **must also** carry `-c model_verbosity=low`. The verbosity default is per-call but unconditional — sticky follow-ups do not re-elevate to Codex' configured default.
- The new prompt file under `/tmp/claudex-prompts/<call_id>/` (a **fresh** `<call_id>` subdirectory, distinct from Invocation A's) again contains the output-discipline language.

## Negative check — never auto-elevated

The orchestrator must **never** set `model_verbosity` to `medium` or `high`, even on a complex subtask. Run, in the same conversation:

```
now in tests/sandboxes/09-verbosity/, design and implement a small TTL-cache module ttl_cache.py with unit tests in test_ttl_cache.py covering insertion, expiry, and overwrite. Do not modify anything outside tests/sandboxes/09-verbosity/.
```

### Expected behaviour

- Phase 1 likely classifies this as `complex` (which may add `-c model_reasoning_effort=high` per scenario 06).
- The Bash call **must still** contain `-c model_verbosity=low` and **must not** contain any other `model_verbosity=` value. Higher reasoning effort does not imply higher verbosity.

## Verify

```bash
# Files Codex was supposed to produce.
ls tests/sandboxes/09-verbosity/
# expect at least: greet.py, test_greet.py, ttl_cache.py, test_ttl_cache.py

# At least one prompt file from each invocation exists, and the most recent ones contain
# the output-discipline language (substance, not exact wording). `ls -t` puts the most recent
# per-call subdirectory on top; each one holds its run's prompt file.
prompt_root="$(cd /tmp/claudex-prompts && pwd -W 2>/dev/null || pwd)"
ls -t "$prompt_root" | head -3
# For each of the three most recent <call_id> subdirectories, open the prompt file inside and confirm:
#   - "no preamble" / "no recap" / "no narration" (any of these phrasings)
#   - "summarise" / "summary" with "5 bullets" or similar cap
#   - "quote" + "errors" + "verbatim"

# Nothing escaped the sandbox.
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

If your client hides the underlying Bash command, look for the orchestrator's Phase 3 transcript output and confirm `-c model_verbosity=low` appears there. If neither the bash call nor the prompt file are visible in your client, this scenario can't be verified there.

## Failure modes to watch for

- `-c model_verbosity=low` missing on **any** of the three invocations → regression on the unconditional rule.
- Prompt file lacks the output-discipline language → the flag alone won't keep summaries useful; both halves must ship together.
- Codex' final message in the transcript is multiple paragraphs of prose for the small Invocation A → either the prompt rule isn't being applied, or `model_verbosity=low` isn't actually being passed (check the Bash call).
- Orchestrator adds a per-subtask `model_verbosity` (e.g. `high` for `complex`) → this is explicitly out of scope; flag and recommend revert.
- Invocation A or B keeps the work in Claude (no `codex exec` call) → either the test-verification step in the prompt was dropped, or the trivial-code carve-out is over-firing. Both halves of the verbosity check require Codex to actually run; without a Codex call there is nothing to inspect.

## Cleanup

```bash
rm -rf tests/sandboxes/09-verbosity
# Per-call subdirectories under /tmp/claudex-prompts/ are intentionally not cleaned —
# the OS temp cleanup handles them and the files are useful for debugging botched runs.
```
