# 05 — sticky mode (follow-up without re-typing `/claudex`)

## Goal

Verify that after `/claudex` has run once, a task-shaped follow-up in the **same conversation** triggers the full orchestration (decompose → delegate → review → report) **without** the user re-typing `/claudex`.

Also verify the negative case: a question-shaped follow-up does **not** kick off Codex.

## Setup

```bash
rm -rf tests/sandboxes/05-sticky
mkdir -p tests/sandboxes/05-sticky
```

## Invocation 1 (priming the session)

Open a fresh Claude Code session and run:

```
/claudex In tests/sandboxes/05-sticky/, create greet.py with a function greet(name) that returns "hello, <name>!". Do not modify anything outside tests/sandboxes/05-sticky/.
```

Watch the orchestrator complete Phase 1–5 normally.

## Invocation 2 (sticky follow-up, no slash command)

In the **same** conversation, send a plain message — no `/claudex` prefix:

```
now also add a shout(name) function in the same file that returns the greeting in upper case
```

## Expected behaviour

- Claude recognises the message as **task-shaped** and silently re-enters the claudex workflow:
  - Phase 1 picks up the prior file scope (`greet.py` in the same sandbox).
  - Phase 3 resolves a fresh per-call subdirectory under `/tmp/claudex-prompts/` (its own `<call_id>`, distinct from the first invocation's), writes a new prompt file there, and runs a fresh `codex exec --sandbox workspace-write --cd tests/sandboxes/05-sticky - < /tmp/claudex-prompts/<call_id>/<slug>-<id>.md` call.
  - Phase 4 reviews the new diff.
  - Phase 5 reports.
- Claude does **not** ask the user "do you want me to use /claudex?". The whole point is that sticky mode is automatic.

## Invocation 3 (question, should NOT trigger Codex)

Still in the same conversation:

```
what does the greet() function return for an empty string?
```

## Expected behaviour

- Claude answers from the diff/file content directly. **No Codex invocation.**
- No new file changes appear in the sandbox.

## Invocation 4 (explicit opt-out, then a fresh task)

```
just answer me directly: how would you add a goodbye function?
```

→ Claude explains in chat, no Codex.

```
ok, now actually implement that goodbye function
```

→ Sticky mode resumes. Codex invocation happens.

## Verify

```bash
cat tests/sandboxes/05-sticky/greet.py
python -c "
import sys; sys.path.insert(0, 'tests/sandboxes/05-sticky')
from greet import greet, shout, goodbye
assert greet('world') == 'hello, world!'
assert shout('world') == 'HELLO, WORLD!'
print('greet, shout, goodbye all importable; greet/shout assertions pass')
"

# nothing outside the sandbox
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

## Failure modes to watch for

- After Invocation 2, Claude asks "should I use /claudex for this?" → sticky mode broken; the prompt rule in `claudex.md` was misinterpreted.
- After Invocation 3 (the question), Claude kicks off a Codex job anyway → over-eager sticky; the "what does X return" pattern was misclassified as task-shaped.
- After Invocation 4's first message ("just answer"), Claude still delegates → the explicit opt-out wasn't respected.

## Cleanup

```bash
rm -rf tests/sandboxes/05-sticky
```
