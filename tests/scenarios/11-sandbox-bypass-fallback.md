# 11 — Sandbox-rejection runtime fallback (macOS/Linux only)

## Goal

Verify Phase 3's runtime fallback for Codex' `patch rejected: writing is blocked by read-only sandbox` error: when a `codex exec` call fails with that exact error string (because the project isn't in `~/.codex/config.toml`'s trust list), the orchestrator retries the **same** call once with `--dangerously-bypass-approvals-and-sandbox` and surfaces a prominent notice in the Phase 5 report telling the user to run `/claudex:setup` to fix it permanently.

**Platform:** macOS/Linux only. On native Windows with Codex ≥ 0.145 the untrusted case may simply never arise (`codex exec` appends the trust entry itself on first run) — Windows-native behaviour is covered by [14-windows-native-sandbox.md](14-windows-native-sandbox.md), not this scenario. Note that newer Codex versions may auto-trust on macOS/Linux too; if the first call unexpectedly succeeds, check `~/.codex/config.toml` for a fresh entry before concluding the fallback is broken.

## Setup

This scenario requires the current project to be **not trusted** by Codex. Either (a) run it on a machine where `~/.codex/config.toml` doesn't have a `[projects."<this-repo-path>"]` entry, or (b) temporarily remove the entry for the test:

```bash
# Optional: stash the existing trust entry for this project.
config_path="${CODEX_HOME:-$HOME/.codex}/config.toml"
cp "$config_path" "$config_path.bak"

project_path="$(pwd)"
echo "current project: $project_path"
# Manually delete the matching [projects."<project_path>"] block from "$config_path" before running.

rm -rf tests/sandboxes/11-bypass
mkdir -p tests/sandboxes/11-bypass
```

## Invocation

```
/claudex In tests/sandboxes/11-bypass/, create a file marker.txt containing the single line: bypass fallback fired. Do not modify anything outside tests/sandboxes/11-bypass/.
```

## Expected behaviour

- **First Codex call** runs with `--sandbox workspace-write` (no bypass flag). On an untrusted project, Codex fails with `patch rejected: writing is blocked by read-only sandbox; rejected by user approval settings` and does not write `marker.txt`.
- The orchestrator detects the substring `patch rejected: writing is blocked by read-only sandbox` in the job's `.out` file and **retries the same `codex exec` call once**, this time with `--dangerously-bypass-approvals-and-sandbox` appended. The prompt file is reused (no new prompt written); the retry gets a fresh `<name>-retry` job name so the first attempt's `.out`/`.exit` stay on disk.
- The retry succeeds and `marker.txt` is created.
- **Phase 5 report** includes a prominent notice along the lines of:
  > Codex sandbox rejected writes for this subtask. Retried with `--dangerously-bypass-approvals-and-sandbox`. Run `/claudex:setup` to permanently trust this project so future runs don't need the bypass.

  This must appear in the report, not be silently swallowed.
- The orchestrator does **not** keep adding the bypass flag for unrelated future subtasks proactively — only the actual sandbox-rejection trigger fires the retry.

## Verify

```bash
# 1. Functional check: the file exists despite the initial sandbox rejection.
cat tests/sandboxes/11-bypass/marker.txt
# expected: bypass fallback fired

# 2. The orchestrator's report must mention the bypass and recommend /claudex:setup.
#    (Read the assistant's Phase 5 report in the Claude Code transcript and confirm.)

# 3. Nothing outside the sandbox should have changed.
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

## Failure modes to watch for

- The first call fails with the sandbox-rejection error and the orchestrator does **not** retry → the fallback isn't wired up. Check Phase 3 "Sandbox trust handling" in [claudex.md](../../plugins/claudex/commands/claudex.md).
- The orchestrator retries on **any** Codex failure, not just sandbox rejection → the fallback is too eager and is masking real bugs. The retry must only fire on the exact `patch rejected: writing is blocked by read-only sandbox` substring.
- The Phase 5 report doesn't mention the bypass → the user has no idea their call ran with sandboxing disabled. Required notice missing — flag as a regression even if the diff is correct.
- The orchestrator preemptively adds `--dangerously-bypass-approvals-and-sandbox` to any Codex call from the start → wrong on **every** platform; the flag is strictly a one-retry fallback. (The old Windows preemptive-bypass steady state died with the Codex 0.145 sandbox fix — see scenario 14.)

## Cleanup

```bash
rm -rf tests/sandboxes/11-bypass

# Restore the trust list if you stashed it during Setup.
config_path="${CODEX_HOME:-$HOME/.codex}/config.toml"
[ -f "$config_path.bak" ] && mv "$config_path.bak" "$config_path"
```
