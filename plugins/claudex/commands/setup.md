---
description: Verify the local Codex CLI is installed and authenticated for use by /claudex.
argument-hint: "[dir]"
allowed-tools: Bash, AskUserQuestion
---

Check that the local Codex CLI is ready for `/claudex` to call.

## Argument parsing

`$ARGUMENTS` may contain one optional directory path. If present, Step 4 checks (and, on consent, trusts) **that** directory instead of the current working directory — useful before multi-repo `/claudex` tasks, where every target repo needs its own trust entry. Verify the directory exists first (`test -d`); if it doesn't, report that and stop. If `$ARGUMENTS` is empty, Step 4 targets the current working directory as before. Steps 1–3 are unaffected by the argument.

## Step 1 — Detect Codex CLI

Run these (in parallel is fine):

```bash
command -v codex
codex --version
```

- If `command -v codex` returns nothing → Codex is **not installed**. Skip to Step 2.
- If `codex --version` succeeds → record the version. Continue to Step 3.

## Step 2 — Offer to install (only if not installed)

Check `command -v npm`. If npm is missing, tell the user to install Node.js + npm first (see https://nodejs.org/) and stop.

If npm is available, use `AskUserQuestion` exactly once with these two options (install first, recommended):

- `Install Codex (Recommended)`
- `Skip for now`

If the user picks install:

```bash
npm install -g @openai/codex
```

Then re-run Step 1. If install failed (e.g. EACCES), surface the error and stop.

If the user skips, tell them how to install manually (`npm install -g @openai/codex`) and stop.

## Step 3 — Check authentication

Run:

```bash
codex login status
```

- If logged in → all good. Report Codex version + auth status and continue to Step 4.
- If not logged in → tell the user to run `codex login` themselves in a terminal (do **not** invoke `codex login` from here — it opens an interactive browser flow that does not work inside Claude Code). Stop and wait for them to confirm before they retry `/claudex`.

## Step 4 — Check project trust (target directory)

On macOS/Linux, Codex' `--sandbox workspace-write` only actually permits writes when the target project (the optional directory argument, or the current working directory) is in the user's trust list at `~/.codex/config.toml` (`[projects."<path>"] trust_level = "trusted"`). Without it, every patch is rejected with `patch rejected: writing is blocked by read-only sandbox`, and `/claudex` ends up forced onto the `--dangerously-bypass-approvals-and-sandbox` runtime fallback — correct, but with sandboxing fully off. Setting the trust entry once removes the need.

**On native Windows the manual trust flow is unnecessary — what matters is the Codex version.** With Codex ≥ 0.145 (verified on 0.145.0), `--sandbox workspace-write` works natively on Windows and `codex exec` appends the workdir's trust entry to `~/.codex/config.toml` by itself on first run, so there is nothing to pre-add here. With **older** Codex CLIs (degradation confirmed through 0.128), `workspace-write` degrades to read-only unconditionally on Windows — trust entries do not help in any TOML path form — and every write-capable `/claudex` call ends up on the `--dangerously-bypass-approvals-and-sandbox` runtime fallback. The fix for that is updating Codex, and Step 4's Windows branch below checks exactly this.

### 4.0 Platform gate

```bash
case "$(uname -s 2>/dev/null)" in
  MINGW*|MSYS*|CYGWIN*) echo "PLATFORM=windows" ;;
  *) echo "PLATFORM=unix" ;;
esac
```

- **`PLATFORM=windows`** → skip 4.1 and 4.2 (with Codex ≥ 0.145, `codex exec` manages the trust entry itself on first run — there is nothing to pre-add, no matter which directory was targeted). Instead, check the installed version against the 0.145 Windows sandbox fix:

  ```bash
  ver="$(codex --version 2>/dev/null | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1)"
  if [ -z "$ver" ]; then
    echo "WIN_SANDBOX=unknown"
  elif [ "$(printf '%s\n' 0.145.0 "$ver" | sort -V | head -1)" = "0.145.0" ]; then
    echo "WIN_SANDBOX=ok version=$ver"
  else
    echo "WIN_SANDBOX=outdated version=$ver"
  fi
  ```

  - `WIN_SANDBOX=ok` → the sandbox works natively on this Windows install; report that in Step 5.
  - `WIN_SANDBOX=outdated` → warn the user: on this Codex version, `workspace-write` degrades to read-only on native Windows and every write-capable `/claudex` call will fall back to `--dangerously-bypass-approvals-and-sandbox` (with a notice per call). Recommend updating: `npm install -g @openai/codex`.
  - `WIN_SANDBOX=unknown` → say the version couldn't be parsed and that `/claudex` will surface a fallback notice if the sandbox turns out to be degraded.

  Then go to Step 5.
- **`PLATFORM=unix`** → continue with 4.1.

### 4.1 Detect & check (single Bash call)

Run the entire detection + grep in **one** Bash invocation so the shell variables survive between the steps. The block prints a structured trailer that the orchestrator parses to decide what to do next:

```bash
# When /claudex:setup got a directory argument, substitute it here; otherwise drop the cd line.
cd "<target-dir>" || { echo "TRUST_RESULT=unknown reason=target_dir_missing"; exit 0; }

config_path="${CODEX_HOME:-$HOME/.codex}/config.toml"
project_path="$(pwd)"

if [ -z "$project_path" ]; then
  echo "TRUST_RESULT=unknown reason=path_detection_failed"
  exit 0
fi

needle="[projects.\"${project_path}\"]"

if grep -Fq "$needle" "$config_path" 2>/dev/null; then
  status=trusted
else
  status=untrusted
fi

# Structured trailer for the orchestrator to parse — one key=value per line so the path
# (which may contain spaces) survives unmolested.
printf 'TRUST_RESULT=%s\n' "$status"
printf 'config_path=%s\n' "$config_path"
printf 'project_path=%s\n' "$project_path"
```

Parse the trailing `TRUST_RESULT=...` line:

- **`TRUST_RESULT=trusted`** → already in the list. Continue to Step 5.
- **`TRUST_RESULT=untrusted`** → continue with 4.2.
- **`TRUST_RESULT=unknown`** → skip the auto-add path; mention in Step 5's summary that you couldn't determine trust state and that the user can add the project manually via `codex` interactive mode.

### 4.2 Ask the user, then (if approved) append

When the result is `untrusted`, ask exactly once with `AskUserQuestion`:

- Question: "The target project (`<project_path from the trailer>`) isn't in your Codex trust list. Without it, `/claudex` will hit Codex' read-only sandbox and fall back to `--dangerously-bypass-approvals-and-sandbox` on every call. Add it to `~/.codex/config.toml` now?"
- Options:
  - `Add to trust list (recommended)`
  - `Skip — I'll handle trust myself`

If the user picks **skip**, note in the Step 5 summary that the project is untrusted and `/claudex` will rely on the bypass-flag fallback until they add it. Stop the trust flow.

If the user picks **add**, append the entry in **one** Bash call. Re-derive `config_path` and `project_path` inside this call (do not trust the previous shell's variables — they are gone). The block also re-checks that no entry exists, so a second `/claudex:setup` run after the first one took effect is safe and idempotent:

```bash
# Same rule as 4.1: substitute the directory argument if one was given; otherwise drop the cd line.
cd "<target-dir>" || { echo "TARGET_DIR_MISSING — no change written"; exit 0; }

config_path="${CODEX_HOME:-$HOME/.codex}/config.toml"
project_path="$(pwd)"

needle="[projects.\"${project_path}\"]"

mkdir -p "$(dirname "$config_path")"
touch "$config_path"

if grep -Fq "$needle" "$config_path" 2>/dev/null; then
  echo "ALREADY_PRESENT — no change written"
else
  # Ensure the file ends with a newline before appending; otherwise the new section header
  # would be glued onto whatever line was last in the file.
  if [ -s "$config_path" ] && [ -n "$(tail -c1 "$config_path")" ]; then
    printf '\n' >> "$config_path"
  fi
  printf '\n%s\ntrust_level = "trusted"\n' "$needle" >> "$config_path"
  echo "APPENDED $needle to $config_path"
fi
```

Read the trailer line. Report `APPENDED ...` (or `ALREADY_PRESENT`) in the Step 5 summary so the user can see exactly what was written.

Do not append the entry without explicit confirmation — modifying user config is exactly the kind of action that requires consent.

## Step 5 — Final summary

Report a single short status block:

```
codex CLI:        <version or "not installed">
auth:             <logged in as <account> | not logged in | unknown>
project trusted:  <yes | no — using --dangerously-bypass-approvals-and-sandbox fallback | auto-managed by codex exec (native Windows, Codex ≥ 0.145) | outdated Codex on native Windows — bypass fallback until updated | unknown>
ready for /claudex: <yes/no>
```

When a directory argument was given, name it on the `project trusted:` line so it's unambiguous which repo was checked.

On native Windows, spell the situation out in one extra sentence below the block. With Codex ≥ 0.145: the sandbox works natively and `codex exec` manages trust entries itself — no action needed. With an older Codex: `workspace-write` degrades to read-only there, every write-capable `/claudex` call will fall back to `--dangerously-bypass-approvals-and-sandbox` with a per-call notice, and updating Codex (`npm install -g @openai/codex`) is the fix.

If `ready` is `no`, list the remaining step(s) the user needs to do.

## Notes

- `/claudex` uses the local Codex CLI's existing auth — there is no separate API key configuration in this plugin.
- If `codex login status` is not a recognised subcommand on the user's installed version, fall back to running `codex --help` and inspecting the output for the right subcommand. Report what you found.
- The Windows version gate reflects observed behaviour: on Codex CLI 0.128, native-Windows `workspace-write` degraded to read-only regardless of trust entries (repo-root, exact `--cd` subdir, lowercased-backslash and exact-case TOML forms all tested, none effective); on Codex CLI 0.145.0 the sandbox works natively and `codex exec` was observed to append the workdir's trust entry itself. The manual trust flow therefore stays unix-only — on Windows it is either unnecessary (≥ 0.145) or ineffective (older).
- Codex ≥ 0.145 may append trust entries automatically on other platforms too. That's fine — 4.1 will simply find the project already trusted and the untrusted case never fires.
