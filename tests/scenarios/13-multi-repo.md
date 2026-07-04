# 13 — Multi-repo delegation (per-repo --cd, trivial exception off)

## Goal

Verify the multi-repo path end to end:

1. **Routing.** A task naming two repos produces one Codex subtask per repo, each invoked with its own `--cd <repo>` — even though each repo's individual change is small enough to pass the trivial-code filter on its own. The trivial exception must **not** fire on multi-repo tasks, and Claude must not edit either repo with Edit/Write itself.
2. **Parallelism.** The two per-repo `codex exec` calls launch in parallel (`run_in_background: true` in one tool batch) — different repos are file-disjoint by definition.
3. **Scope per repo.** Each prompt file carries a files-in-scope list covering only its own repo, with paths relative to that repo's root.
4. **Review per repo.** Phase 4 runs `git -C <repo> status` / `git -C <repo> diff` for each repo (the sandbox repos are their own git repos — the claudex repo's git doesn't see into them), and the Phase 5 report groups entries per repo.
5. **Trust per repo (macOS/Linux).** The freshly-created sandbox repos are (normally) not in the Codex trust list, so the bypass fallback may fire — once per affected repo, and each Phase 5 notice must name **which** repo it fired for and recommend `/claudex:setup <that-repo-path>`. On native Windows this expectation changes: both write-capable calls carry `--dangerously-bypass-approvals-and-sandbox` from the start with the steady-state notice instead (see scenario 14) — no rejection-then-retry pairs, no `/claudex:setup` recommendation.

## Setup

```bash
rm -rf tests/sandboxes/13-multi-repo
for repo in repo-a repo-b; do
  mkdir -p "tests/sandboxes/13-multi-repo/$repo"
  cat > "tests/sandboxes/13-multi-repo/$repo/app.py" <<EOF
def name():
    return "$repo"
EOF
  git -C "tests/sandboxes/13-multi-repo/$repo" init -q
  git -C "tests/sandboxes/13-multi-repo/$repo" add -A
  git -C "tests/sandboxes/13-multi-repo/$repo" commit -qm "init"
done
```

## Invocation

```
/claudex In each of the two repos tests/sandboxes/13-multi-repo/repo-a and tests/sandboxes/13-multi-repo/repo-b, add a VERSION file containing exactly "1.0.0" and extend app.py with a get_version() function that reads and returns the stripped contents of that file. Do not modify anything outside those two directories.
```

Note the per-repo change is deliberately tiny (2 files, fully specified, no test loop) — on a single repo it would legitimately stay in Claude under the trivial-code exception. Producing two Codex calls anyway is exactly what this scenario proves.

## Expected behaviour

- Phase 1 decomposes per repo: two Codex subtasks, no trivial carve-out, no inline Edit/Write on either repo's files by Claude.
- Two prompt files under `/tmp/claudex-prompts/<call_id>/` (same single `call_id` — one `/claudex` call), each with a files-in-scope list naming only `VERSION` and `app.py` relative to its own repo.
- Two `codex exec` calls, launched in parallel as background jobs, each roughly:
  ```bash
  codex exec --sandbox workspace-write --cd tests/sandboxes/13-multi-repo/repo-a -c model_verbosity=low - < /tmp/claudex-prompts/<call_id>/<slug-a>-<id>.md
  codex exec --sandbox workspace-write --cd tests/sandboxes/13-multi-repo/repo-b -c model_verbosity=low - < /tmp/claudex-prompts/<call_id>/<slug-b>-<id>.md
  ```
- **macOS/Linux:** if Codex rejects writes (`patch rejected: writing is blocked by read-only sandbox` — likely, since the sandbox repos were just created and aren't trusted), the bypass retry fires **per affected repo**, and the Phase 5 **Notices** section names each repo individually and recommends `/claudex:setup tests/sandboxes/13-multi-repo/<repo>`.
- **Native Windows:** both `codex exec` calls carry `--dangerously-bypass-approvals-and-sandbox` from the start (see scenario 14) and the Notices section states the steady state — it must **not** recommend `/claudex:setup` for either repo.
- Phase 4 reviews each repo through its own git: `git -C tests/sandboxes/13-multi-repo/repo-a diff` etc.
- The Phase 5 report groups **Delegated to Codex** and **Review notes** per repo, entries prefixed with the repo path.

## Variant B (optional) — repo outside the invocation directory

Create a third repo in the OS temp dir and rerun with all three repos named. This proves `--cd` reaches out-of-tree paths and that Claude doesn't fall back to "Codex can't reach it, I'll edit it myself":

```bash
out_repo="$(cd "${TMPDIR:-/tmp}" && pwd -W 2>/dev/null || pwd)/claudex-scenario-13-repo-c"
rm -rf "$out_repo"; mkdir -p "$out_repo"
printf 'def name():\n    return "repo-c"\n' > "$out_repo/app.py"
git -C "$out_repo" init -q && git -C "$out_repo" add -A && git -C "$out_repo" commit -qm "init"
echo "$out_repo"   # use this absolute path in the invocation
```

Expected: repo-c gets its own `codex exec … --cd <absolute out-of-tree path> …` call like the others; no Claude-side Edit on it.

## Verify

```bash
# Both repos got the change and it works.
for repo in repo-a repo-b; do
  cat "tests/sandboxes/13-multi-repo/$repo/VERSION"        # expect: 1.0.0
  git -C "tests/sandboxes/13-multi-repo/$repo" status --porcelain   # expect: VERSION + app.py modified/added, nothing else
  (cd "tests/sandboxes/13-multi-repo/$repo" && python -c "import app; print(app.get_version())")  # expect: 1.0.0
done

# Exactly one call_id dir with two (Variant B: three) prompt files, scoped per repo.
prompt_root="$(cd /tmp/claudex-prompts && pwd -W 2>/dev/null || pwd)"
ls -t "$prompt_root" | head -1
# Open both prompt files: each names only its own repo's files (relative paths), not the sibling repo's.

# Nothing escaped into the claudex repo itself.
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

In the transcript, confirm: no Edit/Write tool call touching either sandbox repo's files; both `codex exec` calls launched with `run_in_background: true` in one batch; any bypass notice names its repo.

## Failure modes to watch for

- Claude applies the change itself with Edit/Write "because it's trivial" → the trivial exception fired on a multi-repo task; hard regression.
- One combined Codex call for both repos (single `--cd`, or none) → per-repo decomposition missing; also breaks the sandbox isolation between the repos.
- The two calls run sequentially without a stated reason → different repos are file-disjoint by definition; sequential here means the parallelism rule regressed.
- A prompt's files-in-scope list names the other repo's files, or uses paths relative to the claudex repo instead of the `--cd` root → scope rule regression.
- Bypass fallback fires but the notice doesn't say which repo, or fires once "for the task" instead of per call → per-repo trust reporting regressed (macOS/Linux).
- On native Windows: a sandboxed first attempt followed by a rejection-then-retry pair → the platform detection didn't fire; see scenario 14.
- Variant B: Claude edits the out-of-tree repo itself, claiming Codex can't reach it → `--cd` takes any absolute path; this is the exact failure mode the multi-repo rules exist to prevent.
- Phase 4 runs only top-level `git status` and reports "no changes" → per-repo review (`git -C`) missing.

## Cleanup

```bash
rm -rf tests/sandboxes/13-multi-repo
rm -rf "${TMPDIR:-/tmp}/claudex-scenario-13-repo-c" 2>/dev/null
# If you accepted trust entries for the sandbox repos during the run, remove their
# [projects.'…13-multi-repo…'] sections from ~/.codex/config.toml again.
# Per-call subdirectories under /tmp/claudex-prompts/ are intentionally not cleaned.
```
