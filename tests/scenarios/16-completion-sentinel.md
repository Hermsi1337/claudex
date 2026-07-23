# 16 — Completion via sentinel files, not notifications

## Goal

Verify Phase 3's sentinel protocol end to end: every Codex job is launched in the background with its output redirected to `<name>.out` and an exit-code sentinel `<name>.exit`, a watcher (`until`-loop background call) provides the wake-up, and the orchestrator collects results from the sentinel files instead of waiting on the Codex job's own background-task notification. This is the guard against the "orchestrator waits forever on a Codex call that already finished" failure — harness completion notifications go missing often enough (native Windows, subagent contexts) that they must never be the thing the orchestrator sleeps on.

**Platform:** any.

## Setup

```bash
rm -rf tests/sandboxes/16-sentinel
mkdir -p tests/sandboxes/16-sentinel
```

## Invocation

```
/claudex In tests/sandboxes/16-sentinel/, create fizzbuzz.py with a function fizzbuzz(n: int) -> str returning "fizz" for multiples of 3, "buzz" for multiples of 5, "fizzbuzz" for both, and str(n) otherwise. Do not modify anything outside tests/sandboxes/16-sentinel/.
```

## Expected behaviour

- **Launch:** the `codex exec` Bash call has `run_in_background: true` and ends with `- < <call-dir>/<name>.md > <call-dir>/<name>.out 2>&1; echo $? > <call-dir>/<name>.exit`. No Codex output streams into the transcript — it all lands in `<name>.out`.
- **Watcher:** immediately after the launch (same turn or the next), a second background Bash call runs an `until`-loop that exits when `<name>.exit` exists, with a bounded iteration count (the claudex.md template caps at ~30 min) and a `WATCHER-TIMEOUT` / `ALL-SENTINELS-PRESENT` trailer.
- **No idle waiting:** the orchestrator never ends a turn "waiting for the Codex job" without that watcher armed, and it never busy-polls in a tight loop of instant checks.
- **Collection:** after the wake-up, the orchestrator reads the exit code from `<name>.exit` and Codex' final summary from `<name>.out` (Read tool or a `tail`), then proceeds to Phase 4 review — even if the harness still lists the Codex background task as running.
- **Phase 5:** normal report; a "process lingered after completing" notice is acceptable if the `.exit` file was late, per the grace rule.

## Verify

```bash
# 1. Functional check.
python -c "
import sys; sys.path.insert(0, 'tests/sandboxes/16-sentinel')
from fizzbuzz import fizzbuzz
assert fizzbuzz(3) == 'fizz' and fizzbuzz(5) == 'buzz' and fizzbuzz(15) == 'fizzbuzz' and fizzbuzz(7) == '7'
print('fizzbuzz: behaves as expected')
"

# 2. Sentinel triple exists in the newest per-call dir: prompt, output, exit code.
call_dir="$(ls -td "$(cd /tmp/claudex-prompts && pwd -W 2>/dev/null || pwd)"/* | head -1)"
ls "$call_dir"
# expected: <name>.md, <name>.out, <name>.exit for the subtask
cat "$call_dir"/*.exit
# expected: 0

# 3. Transcript check: background launch with the redirects; a separate until-loop watcher
#    call; results read from the sentinel files; no turn ended waiting without a watcher.

# 4. Nothing outside the sandbox changed.
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

## Failure modes to watch for

- The `codex exec` call has no `> <name>.out 2>&1; echo $? > <name>.exit` suffix → sentinel protocol regressed at the launch step; nothing downstream can work.
- No watcher call after the launch, the orchestrator just ends its turn → the exact "sub agents fell asleep" bug this protocol exists to prevent; if the completion notification is lost, this session hangs forever.
- The orchestrator polls the background task's status/output in a rapid series of foreground checks instead of arming one watcher → token-burning busy-wait; regression against the "one watcher per batch" rule.
- The orchestrator waits for the Codex Bash task to be reported complete even though `.exit` exists → sentinel files must win over harness task status.
- Watcher with no iteration cap (`until [ -f … ]; do sleep 5; done` and nothing else) → an unbounded watcher can hang exactly like the job it watches; the cap + `WATCHER-TIMEOUT` trailer are required.

## Cleanup

```bash
rm -rf tests/sandboxes/16-sentinel
# Per-call subdirectories under /tmp/claudex-prompts/ are intentionally not cleaned.
```
