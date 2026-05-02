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

- If logged in → all good. Report Codex version + auth status and stop.
- If not logged in → tell the user to run `codex login` themselves in a terminal (do **not** invoke `codex login` from here — it opens an interactive browser flow that does not work inside Claude Code). Stop and wait for them to confirm before they retry `/claudex`.

## Step 4 — Final summary

Report a single short status block:

```
codex CLI:    <version or "not installed">
auth:         <logged in as <account> | not logged in | unknown>
ready for /claudex: <yes/no>
```

If `ready` is `no`, list the remaining step(s) the user needs to do.

## Notes

- `/claudex` uses the local Codex CLI's existing auth — there is no separate API key configuration in this plugin.
- If `codex login status` is not a recognised subcommand on the user's installed version, fall back to running `codex --help` and inspecting the output for the right subcommand. Report what you found.
