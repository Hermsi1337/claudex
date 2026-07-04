---
description: Verify the local Codex CLI is installed and authenticated for use by /claudex.
argument-hint: ""
allowed-tools: Bash, AskUserQuestion
---

Check that the local Codex CLI is ready for `/claudex` to call.

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

## Step 4 — Check project trust (current working directory)

On macOS/Linux, Codex' `--sandbox workspace-write` only actually permits writes when the current project is in the user's trust list at `~/.codex/config.toml` (`[projects."<path>"] trust_level = "trusted"`). Without it, every patch is rejected with `patch rejected: writing is blocked by read-only sandbox`, and `/claudex` ends up forced onto the `--dangerously-bypass-approvals-and-sandbox` runtime fallback — correct, but with sandboxing fully off. Setting the trust entry once removes the need.

**On native Windows this whole step is moot.** With Codex CLI 0.128, `--sandbox workspace-write` degrades to read-only unconditionally (the session header reports `sandbox: read-only`) — trust-list entries do **not** enable writes, in any TOML path form. `/claudex` therefore runs every write-capable Codex call with `--dangerously-bypass-approvals-and-sandbox` on Windows as the expected steady state.

### 4.0 Platform gate

```bash
case "$(uname -s 2>/dev/null)" in
  MINGW*|MSYS*|CYGWIN*) echo "PLATFORM=windows" ;;
  *) echo "PLATFORM=unix" ;;
esac
```

- **`PLATFORM=windows`** → skip 4.1 and 4.2 entirely (do not offer to add a trust entry — it has no effect). Go to Step 5 and report the Windows steady state there.
- **`PLATFORM=unix`** → continue with 4.1.

### 4.1 Detect & check (single Bash call)

Run the entire detection + grep in **one** Bash invocation so the shell variables survive between the steps. The block prints a structured trailer that the orchestrator parses to decide what to do next:

```bash
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

- Question: "The current project (`<project_path from the trailer>`) isn't in your Codex trust list. Without it, `/claudex` will hit Codex' read-only sandbox and fall back to `--dangerously-bypass-approvals-and-sandbox` on every call. Add it to `~/.codex/config.toml` now?"
- Options:
  - `Add to trust list (recommended)`
  - `Skip — I'll handle trust myself`

If the user picks **skip**, note in the Step 5 summary that the project is untrusted and `/claudex` will rely on the bypass-flag fallback until they add it. Stop the trust flow.

If the user picks **add**, append the entry in **one** Bash call. Re-derive `config_path` and `project_path` inside this call (do not trust the previous shell's variables — they are gone). The block also re-checks that no entry exists, so a second `/claudex:setup` run after the first one took effect is safe and idempotent:

```bash
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
project trusted:  <yes | no — using --dangerously-bypass-approvals-and-sandbox fallback | n/a on native Windows — sandbox unavailable, /claudex always uses --dangerously-bypass-approvals-and-sandbox | unknown>
ready for /claudex: <yes/no>
```

On native Windows, spell the situation out in one extra sentence below the block: Codex' sandbox is unavailable there (workspace-write degrades to read-only regardless of trust entries), so every write-capable `/claudex` Codex call runs with `--dangerously-bypass-approvals-and-sandbox`. The user should know this is by necessity, not an accident.

If `ready` is `no`, list the remaining step(s) the user needs to do.

## Notes

- `/claudex` uses the local Codex CLI's existing auth — there is no separate API key configuration in this plugin.
- If `codex login status` is not a recognised subcommand on the user's installed version, fall back to running `codex --help` and inspecting the output for the right subcommand. Report what you found.
- The Windows-is-moot rule reflects observed behaviour of Codex CLI 0.128 on native Windows: repo-root trust entries, exact `--cd` subdir entries, lowercased-backslash and exact-case TOML forms were all tested and none enabled workspace-write. If a future Codex version fixes sandboxing on Windows, re-enable the trust flow for it.
