# 14 — Native Windows: preemptive sandbox bypass (Windows only)

## Goal

Verify Phase 3's native-Windows rule: because Codex CLI 0.128 degrades `--sandbox workspace-write` to read-only unconditionally on native Windows (trust-list entries do not help), the orchestrator must detect the platform and add `--dangerously-bypass-approvals-and-sandbox` to every write-capable `codex exec` call **from the start** — no doomed sandboxed first attempt, no wasted round-trip — and surface a prominent steady-state notice in the Phase 5 report.

**Platform:** native Windows (Git Bash) only. On macOS/Linux the sandbox works for trusted projects and the bypass flag must stay a runtime fallback — see [11-sandbox-bypass-fallback.md](11-sandbox-bypass-fallback.md).

## Setup

```bash
rm -rf tests/sandboxes/14-winbypass
mkdir -p tests/sandboxes/14-winbypass
```

No trust-list preparation is needed — the whole point is that trust entries are irrelevant on native Windows. Run with or without a `[projects...]` entry for this repo; the expected behaviour is identical.

## Invocation

```
/claudex In tests/sandboxes/14-winbypass/, create a file marker.txt containing the single line: windows bypass steady state. Do not modify anything outside tests/sandboxes/14-winbypass/.
```

## Expected behaviour

- **Phase 3 platform detection:** the per-call prompt-directory resolution returns a Windows drive-letter path (e.g. `C:/Users/<user>/AppData/Local/Temp/claudex-prompts/<call_id>`). The orchestrator treats this as native Windows.
- **Exactly one `codex exec` call** for the subtask, and it includes `--dangerously-bypass-approvals-and-sandbox` from the start:

  ```
  codex exec --sandbox workspace-write --cd tests/sandboxes/14-winbypass -c model_verbosity=low --dangerously-bypass-approvals-and-sandbox - < <resolved-call-dir>/<slug>-<id>.md
  ```

  Inspect the transcript: there must be **no** initial sandboxed attempt that fails with `patch rejected: writing is blocked by read-only sandbox` followed by a retry. One call, flag present, done.
- The call succeeds and `marker.txt` is created.
- **Phase 5 report** includes a prominent notice along the lines of:
  > Native Windows: Codex' sandbox is unavailable (workspace-write degrades to read-only), so Codex ran with `--dangerously-bypass-approvals-and-sandbox`. This is the expected steady state on Windows — trust-list entries do not help.

  The notice must **not** tell the user to run `/claudex:setup` to trust the project — trusting can't fix this on Windows.

## Verify

```bash
# 1. Functional check.
cat tests/sandboxes/14-winbypass/marker.txt
# expected: windows bypass steady state

# 2. Transcript check: exactly one codex exec call for the subtask, bypass flag present
#    from the start, no sandbox-rejection-then-retry pair.

# 3. Report check: the Phase 5 Notices section explains the Windows steady state and
#    does NOT recommend /claudex:setup as a fix.

# 4. Nothing outside the sandbox should have changed.
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

## Failure modes to watch for

- Two `codex exec` calls for the same subtask (sandboxed attempt → rejection → bypass retry) → the platform detection didn't fire; the orchestrator paid the wasted round-trip this rule exists to eliminate. Check the "Native Windows" rule under "Sandbox trust handling" in [claudex.md](../../plugins/claudex/commands/claudex.md).
- The bypass flag is missing and the call fails with `patch rejected: writing is blocked by read-only sandbox` with no recovery → both the Windows rule and the runtime fallback are broken.
- No Phase 5 notice → the user has no idea their call ran with sandboxing disabled. Required notice missing — flag as a regression even if the diff is correct.
- The notice recommends `/claudex:setup` to trust the project → wrong advice on Windows; trust entries don't enable writes there.

## Cleanup

```bash
rm -rf tests/sandboxes/14-winbypass
```
