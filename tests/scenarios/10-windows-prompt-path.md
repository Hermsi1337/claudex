# 10 — Windows prompt-path resolution

## Goal

Verify that on Windows Git Bash (MSYS / MINGW / Cygwin) the orchestrator resolves the prompt directory to the underlying Windows path before writing the prompt file, so the Write tool and the Bash `<` redirect agree on file location. Without this, `/claudex` on Windows fails with `bash: /tmp/claudex-prompts/<call_id>/<slug>.md: No such file or directory` because the Write tool drops `/tmp/...` onto the current drive (`D:/tmp/...`) while Bash's `/tmp` maps to `$TMPDIR` (typically under `C:/Users/...`).

This scenario also incidentally exercises the **per-call subdirectory** layout — every `/claudex` call resolves into its own `<call_id>` directory under the prompt root, so concurrent sessions don't share files.

This scenario is a **no-op on macOS / Linux** — the resolution still runs but produces a path under `/tmp/claudex-prompts/` unchanged. Run it on Windows to verify the fix; on other platforms it should still pass the verification because the same code path is exercised.

## Setup

```bash
rm -rf tests/sandboxes/10-winpath
mkdir -p tests/sandboxes/10-winpath
```

## Invocation

```
/claudex In tests/sandboxes/10-winpath/, create a file note.txt containing the single line: prompt path resolved. Do not modify anything outside tests/sandboxes/10-winpath/.
```

(One-file Codex job — small enough that decomposition is irrelevant; the focus is on prompt-file plumbing.)

## Expected behaviour

- **Phase 3, before any per-subtask work:** the orchestrator runs (or equivalent) the resolution command:
  ```bash
  mkdir -p /tmp/claudex-prompts \
    && call_id="$(date +%Y%m%d-%H%M%S)-$(LC_ALL=C tr -dc 'a-f0-9' </dev/urandom | head -c 6)" \
    && call_dir="/tmp/claudex-prompts/$call_id" \
    && mkdir -p "$call_dir" \
    && (cd "$call_dir" && pwd -W 2>/dev/null || pwd)
  ```
  The output is captured. On Windows Git Bash it should look like `C:/Users/<user>/AppData/Local/Temp/claudex-prompts/<call_id>`. On Mac/Linux it stays `/tmp/claudex-prompts/<call_id>`.
- The Write-tool call uses that resolved absolute path as the directory, e.g.
  - Windows: `Write` to `C:/Users/<user>/AppData/Local/Temp/claudex-prompts/<call_id>/winpath-<id>.md`
  - Mac/Linux: `Write` to `/tmp/claudex-prompts/<call_id>/winpath-<id>.md`
- The Bash `codex exec` call uses **the same resolved path** in the `<` redirect, not a hard-coded `/tmp/claudex-prompts/...`. (Either the resolved absolute path, or `/tmp/claudex-prompts/<call_id>/<file>.md` since Bash's mapping resolves to the same physical file — but on Windows specifically the Write tool needs the resolved path.)
- The Codex call succeeds and `note.txt` is created with the requested content.

## Verify

```bash
# 1. Find the resolved prompt root and the most recent per-call subdirectory.
prompt_root="$(cd /tmp/claudex-prompts && pwd -W 2>/dev/null || pwd)"
latest_call="$(ls -1t "$prompt_root" 2>/dev/null | head -1)"
echo "resolved prompt root: $prompt_root"
echo "latest call dir:      $latest_call"

# 2. The orchestrator should have written its prompt file inside the latest <call_id> dir.
#    On Windows specifically, the file MUST exist under the Windows path — not under D:/tmp/claudex-prompts/.
if [ -n "$latest_call" ] && ls "$prompt_root/$latest_call"/*.md >/dev/null 2>&1; then
  echo "ok: prompt file present in $prompt_root/$latest_call"
else
  echo "NO PROMPT FILE FOUND under resolved path — regression"
fi

# 3. On Windows only: confirm the orchestrator did NOT silently write to the drive-relative location.
case "$(uname -s 2>/dev/null)" in
  MINGW*|MSYS*|CYGWIN*)
    if [ -d "/$(pwd | cut -c1)/tmp/claudex-prompts" ]; then
      # Look for any winpath-* file under any subdirectory of the drive-relative tree.
      leaked="$(find "/$(pwd | cut -c1)/tmp/claudex-prompts" -name 'winpath-*.md' 2>/dev/null | head -1)"
      if [ -n "$leaked" ]; then
        echo "REGRESSION: prompt file leaked to drive-relative /tmp ($leaked)"
      else
        echo "ok: no leak to drive-relative /tmp"
      fi
    fi
    ;;
esac

# 4. Functional check on the actual task output.
cat tests/sandboxes/10-winpath/note.txt
# expected: prompt path resolved

# 5. Nothing outside the sandbox should have changed.
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

## Failure modes to watch for

- The orchestrator skips the resolution step and writes to a literal `/tmp/claudex-prompts/...` path on Windows → the Bash redirect fails with "No such file or directory" referencing the same path, even though the Write succeeded. This is the original regression this scenario exists to catch.
- The resolution is computed but the orchestrator forgets to use it for the Write tool (only uses it in the Bash redirect, or vice versa) → file lands in one place, redirect reads from another. Same observable failure as above.
- The orchestrator skips the per-call subdirectory and writes the prompt file directly into `/tmp/claudex-prompts/` → concurrent sessions will collide. Verify by re-running this scenario twice in quick succession; each run must produce its own `<call_id>` directory under the prompt root.
- The orchestrator decides this is a "trivial" task and never writes a prompt file at all → that's a different regression; see scenario 01. This scenario assumes a Codex round-trip is happening.

## Cleanup

```bash
rm -rf tests/sandboxes/10-winpath
```
