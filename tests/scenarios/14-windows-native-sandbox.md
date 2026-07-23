# 14 — Native Windows: sandboxed writes, no bypass flag (Windows only)

## Goal

Verify the post-0.145 native-Windows behaviour: with Codex CLI ≥ 0.145, `--sandbox workspace-write` works natively on Windows, so a write-capable `/claudex` subtask must run **fully sandboxed** — no `--dangerously-bypass-approvals-and-sandbox` anywhere, no trust-list preparation, no Windows steady-state notice. This scenario replaces the old "preemptive sandbox bypass" expectation, which was correct only for Codex ≤ 0.128 (where Windows `workspace-write` degraded to read-only regardless of trust entries).

**Platform:** native Windows (Git Bash) only. **Codex version:** ≥ 0.145 (`codex --version`). On an older Codex this scenario does not apply — expect the scenario-11-style rejection-then-retry fallback instead, with a notice recommending a Codex update rather than `/claudex:setup`.

## Setup

```bash
rm -rf tests/sandboxes/14-winsandbox
mkdir -p tests/sandboxes/14-winsandbox
codex --version   # must be >= 0.145
```

No trust-list preparation — part of what this scenario proves is that none is needed: `codex exec` appends the workdir's trust entry to `~/.codex/config.toml` itself on first run.

## Invocation

```
/claudex In tests/sandboxes/14-winsandbox/, create a file marker.txt containing the single line: windows native sandbox. Do not modify anything outside tests/sandboxes/14-winsandbox/.
```

## Expected behaviour

- **Phase 3 platform detection** still happens (the per-call prompt-directory resolution returns a drive-letter path), but it does **not** change the call shape.
- **Exactly one `codex exec` call** for the subtask, fully sandboxed and with the sentinel redirects:

  ```
  codex exec --sandbox workspace-write --cd tests/sandboxes/14-winsandbox -c model_verbosity=low - < <call-dir>/<name>.md > <call-dir>/<name>.out 2>&1; echo $? > <call-dir>/<name>.exit
  ```

  Inspect the transcript: `--dangerously-bypass-approvals-and-sandbox` must not appear, and there must be no rejection-then-retry pair.
- The call succeeds, `marker.txt` is created, and the session header inside `<name>.out` reports `sandbox: workspace-write` (not `read-only`).
- **Phase 5 report** carries **no** sandbox notice — no steady-state warning, no `/claudex:setup` recommendation, no update recommendation. (A lingered/stalled sentinel notice is unrelated and allowed.)

## Verify

```bash
# 1. Functional check.
cat tests/sandboxes/14-winsandbox/marker.txt
# expected: windows native sandbox

# 2. The job's .out shows a working sandbox.
grep -m1 'sandbox:' "$(ls -td "$(cd /tmp/claudex-prompts && pwd -W 2>/dev/null || pwd)"/* | head -1)"/*.out
# expected: sandbox: workspace-write [...]

# 3. Transcript check: exactly one codex exec call, no bypass flag, sentinel redirects present.

# 4. Codex auto-added the trust entry (informational — clean it up below).
grep -n '14-winsandbox' "${CODEX_HOME:-$HOME/.codex}/config.toml"

# 5. Nothing outside the sandbox changed.
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

## Failure modes to watch for

- `--dangerously-bypass-approvals-and-sandbox` on the first call → the retired preemptive-bypass behaviour regressed; check "Sandbox trust handling" in [claudex.md](../../plugins/claudex/commands/claudex.md).
- Rejection-then-retry pair on Codex ≥ 0.145 → the Windows sandbox fix isn't in effect on this machine after all; capture `codex --version` and the session header from `<name>.out` and investigate before blaming the orchestrator.
- A Phase 5 notice recommending `/claudex:setup` on Windows → wrong remedy; the unix trust flow doesn't apply here on any Codex version.

## Cleanup

```bash
rm -rf tests/sandboxes/14-winsandbox
# Remove the auto-added [projects.'…14-winsandbox…'] section from ~/.codex/config.toml.
# Per-call subdirectories under /tmp/claudex-prompts/ are intentionally not cleaned.
```
