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

Codex' `--sandbox workspace-write` only actually permits writes when the target project (the optional directory argument, or the current working directory) is in the user's trust list at `~/.codex/config.toml` (`[projects.'<path>'] trust_level = "trusted"` or the platform-equivalent form). Without it, every patch is rejected with `patch rejected: writing is blocked by read-only sandbox`, and `/claudex` ends up forced onto the `--dangerously-bypass-approvals-and-sandbox` runtime fallback — correct, but with sandboxing fully off. Setting the trust entry once removes the need.

### 4.1 Detect & check (single Bash call)

Run the entire detection + grep in **one** Bash invocation so the shell variables survive between the steps. The block prints a structured trailer that the orchestrator parses to decide what to do next:

```bash
# When /claudex:setup got a directory argument, substitute it here; otherwise drop the cd line.
cd "<target-dir>" || { echo "TRUST_RESULT=unknown reason=target_dir_missing"; exit 0; }

config_path="${CODEX_HOME:-$HOME/.codex}/config.toml"

case "$(uname -s 2>/dev/null)" in
  MINGW*|MSYS*|CYGWIN*)
    # Windows native: lowercase backslash form, e.g. d:\develop\foo\bar
    project_path="$(cygpath -w "$(pwd)" 2>/dev/null | tr '[:upper:]' '[:lower:]')"
    quote="'"  # TOML literal string — backslashes preserved as-is
    ;;
  *)
    project_path="$(pwd)"
    quote='"'  # TOML basic string — forward-slash paths need no escaping
    ;;
esac

if [ -z "$project_path" ]; then
  echo "TRUST_RESULT=unknown reason=path_detection_failed"
  exit 0
fi

needle="[projects.${quote}${project_path}${quote}]"

if grep -Fq "$needle" "$config_path" 2>/dev/null; then
  status=trusted
else
  status=untrusted
fi

# Structured trailer for the orchestrator to parse — one key=value per line so the path
# (which may contain spaces or backslashes) survives unmolested.
printf 'TRUST_RESULT=%s\n' "$status"
printf 'config_path=%s\n' "$config_path"
printf 'quote=%s\n' "$quote"
printf 'project_path=%s\n' "$project_path"
```

Parse the trailing `TRUST_RESULT=...` line:

- **`TRUST_RESULT=trusted`** → already in the list. Continue to Step 5.
- **`TRUST_RESULT=untrusted`** → continue with 4.2.
- **`TRUST_RESULT=unknown`** → on Windows, this means `cygpath` is missing (rare). Skip the auto-add path; mention in Step 5's summary that you couldn't determine trust state and that the user can add the project manually via `codex` interactive mode.

### 4.2 Ask the user, then (if approved) append

When the result is `untrusted`, ask exactly once with `AskUserQuestion`:

- Question: "The target project (`<project_path from the trailer>`) isn't in your Codex trust list. Without it, `/claudex` will hit Codex' read-only sandbox and fall back to `--dangerously-bypass-approvals-and-sandbox` on every call. Add it to `~/.codex/config.toml` now?"
- Options:
  - `Add to trust list (recommended)`
  - `Skip — I'll handle trust myself`

If the user picks **skip**, note in the Step 5 summary that the project is untrusted and `/claudex` will rely on the bypass-flag fallback until they add it. Stop the trust flow.

If the user picks **add**, append the entry in **one** Bash call. Re-derive `config_path`, `project_path`, and `quote` inside this call (do not trust the previous shell's variables — they are gone). The block also re-checks that no entry exists, so a second `/claudex:setup` run after the first one took effect is safe and idempotent:

```bash
# Same rule as 4.1: substitute the directory argument if one was given; otherwise drop the cd line.
cd "<target-dir>" || { echo "TARGET_DIR_MISSING — no change written"; exit 0; }

config_path="${CODEX_HOME:-$HOME/.codex}/config.toml"

case "$(uname -s 2>/dev/null)" in
  MINGW*|MSYS*|CYGWIN*)
    project_path="$(cygpath -w "$(pwd)" 2>/dev/null | tr '[:upper:]' '[:lower:]')"
    quote="'"
    ;;
  *)
    project_path="$(pwd)"
    quote='"'
    ;;
esac

needle="[projects.${quote}${project_path}${quote}]"

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
project trusted:  <yes | no — using --dangerously-bypass-approvals-and-sandbox fallback | unknown>
ready for /claudex: <yes/no>
```

When a directory argument was given, name it on the `project trusted:` line so it's unambiguous which repo was checked.

If `ready` is `no`, list the remaining step(s) the user needs to do.

## Notes

- `/claudex` uses the local Codex CLI's existing auth — there is no separate API key configuration in this plugin.
- If `codex login status` is not a recognised subcommand on the user's installed version, fall back to running `codex --help` and inspecting the output for the right subcommand. Report what you found.
- `cygpath` ships with Git Bash / MSYS2; if it's missing on a Windows install, fall back to `pwd` for the trust-check path and warn the user that the auto-format may not match what Codex stores. Manual trust setup via `codex` interactive mode in the project directory remains an option.
