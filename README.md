# claudex

Claude Code plugin that turns Claude into an **orchestrator** and delegates concrete implementation work to **OpenAI Codex** (running locally via the `codex` CLI).

The idea: Claude is great at conceptual work, decomposition and review. Codex is fast at well-scoped implementation tasks. With `claudex`, you stay in Claude Code, hand it a task, and Claude:

1. Splits the task into subtasks.
2. Delegates code-changing subtasks to Codex (in parallel where it's safe).
3. Delegates broad codebase research to Codex in read-only mode — instead of spawning expensive Claude subagents.
4. Handles documentation/conceptual subtasks itself.
5. Reviews Codex' diff.
6. Reports back with what changed and what to do next.

Inspired by [`openai/codex-plugin-cc`](https://github.com/openai/codex-plugin-cc), but with the inverse flow: that plugin uses Codex primarily for code review and bug rescue. `claudex` is **Claude → Codex**, not the other way around.

## Requirements

- [Claude Code](https://claude.com/claude-code) (CLI, IDE extension, or desktop)
- [OpenAI Codex CLI](https://github.com/openai/codex) **≥ 0.128** installed and authenticated locally
  - `npm install -g @openai/codex`
  - `codex login`
  - The plugin uses `codex exec --sandbox workspace-write [--cd <dir>] [--model <name>] - < <prompt-dir>/<id>.md`. Prompts are always fed via stdin from a file (sidesteps shell-escaping bugs on prompts that contain backticks, `$`, or quotes). Each `/claudex` call writes into its own per-call subdirectory `/tmp/claudex-prompts/<call_id>/` (or the Windows-resolved equivalent under `%TEMP%\claudex-prompts\<call_id>\` on Git Bash) so concurrent Claude Code sessions don't pile prompt files on top of each other. `call_id` is timestamped (`20260509-171152-89c895`), so `ls -t /tmp/claudex-prompts/` shows the most recent call on top. Older Codex versions had a `--full-auto` preset that this plugin no longer relies on.
  - **Project trust (macOS/Linux):** Codex' `--sandbox workspace-write` only actually permits writes when the project is in the user's `~/.codex/config.toml` trust list (a `[projects."<path>"] trust_level = "trusted"` entry). `/claudex:setup` checks the current project and offers to add it for you. Without trust, Codex rejects every patch with `patch rejected: writing is blocked by read-only sandbox`; in that case `/claudex` retries the call once with `--dangerously-bypass-approvals-and-sandbox` and surfaces a notice telling you to run `/claudex:setup` to fix it permanently.
  - **Native Windows:** Codex' sandbox is effectively unavailable — `workspace-write` degrades to read-only no matter what's in the trust list (confirmed on Codex CLI 0.128). `/claudex` detects Windows and runs every write-capable Codex call with `--dangerously-bypass-approvals-and-sandbox` from the start, with a notice in each report. See "Sandbox & trust" below.
- macOS, Linux, or Windows (Git Bash on Windows is supported directly; WSL works too)

The plugin shells out to `codex exec` and uses your existing Codex auth — no separate API key configuration.

## Install

In Claude Code:

```
/plugin marketplace add Hermsi1337/claudex
/plugin install claudex@claudex
```

Then verify your Codex CLI is ready:

```
/claudex:setup
```

## Usage

### Simple case

```
/claudex add a healthcheck endpoint at /api/health
```

Claude reads the relevant files, sends one self-contained prompt to Codex, then reviews the diff. For genuinely trivial single-edit changes (rename a symbol with a known target, bump a version pin, fix a typo) Claude applies the change directly without invoking Codex — see the table below.

### Complex case (parallel + sequential)

```
/claudex add user signup with email verification, write API docs, and add integration tests
```

Claude splits this into subtasks, identifies file overlaps, and runs them with the right concurrency:

- Signup implementation (Codex) ‖ API docs (Claude itself) — parallel, file-disjoint
- Integration tests (Codex) — sequential, after the signup code lands

Then Claude reviews everything Codex produced and reports back.

### Multi-repo case

```
/claudex bump the healthcheck timeout to 10s in D:/repos/service-a, D:/repos/service-b and D:/repos/service-c
```

Each repo becomes its own Codex subtask, invoked with `--cd <repo>` — which works for any absolute path, so the repos don't have to live under the directory you started Claude Code in. Different repos never share files, so the per-repo jobs run in parallel. Claude never edits foreign repos itself, and the trivial-edit shortcut is disabled for multi-repo tasks — repeated mechanical edits across repos always go to Codex. Context in a foreign repo is gathered via a read-only research run instead of Claude reading the repo, and review happens per repo (`git -C <repo> diff`).

On macOS/Linux, each target repo needs its own entry in your Codex trust list: run `/claudex:setup <repo-path>` once per repo, or accept the per-repo sandbox-bypass fallback (with a notice naming the repo) until you do. On native Windows the trust list is irrelevant — every write-capable call runs with the preemptive bypass anyway (see "Sandbox & trust" below).

### Override the model

```
/claudex --model gpt-5.6-codex add caching to the user lookup
```

Without `--model`, Codex' own default is used (so you always get its latest model).

### Override the reasoning effort

```
/claudex --effort high refactor the rate limiter to use a token-bucket algorithm
/claudex --effort xhigh design and implement a distributed leader election protocol
/claudex --effort low rename FooManager to FooRegistry across the repo
```

`--effort <low|medium|high|xhigh>` forces that level for **every** Codex subtask in this call and bypasses the auto-classification described below. Effort ordering, lowest to highest: `low < medium < high < xhigh`. Codex' own guidance: `medium` is the everyday interactive default; `high` / `xhigh` are for the hardest, long-running autonomous work. Without `--effort`, Claude decides per subtask: complex ones get escalated to `high`, standard ones run at whatever your `~/.codex/config.toml` says.

### Sticky follow-ups

After your first `/claudex` in a conversation, follow-up task-shaped requests in the same chat are routed through the same orchestrator automatically — no need to retype `/claudex`:

```
/claudex add a healthcheck endpoint at /api/health
… (Claude delegates, reviews, reports)

now also add a /api/version endpoint that returns the package version
… (Claude does the same workflow without you typing /claudex again)
```

Questions about the diff or the codebase don't trigger a Codex run; only sentence-shaped task requests do. To turn it off explicitly: "just answer me directly" or "no more delegating".

## What gets delegated to Codex

| Subtask kind | Handled by |
| --- | --- |
| Implementing a feature | Codex |
| Refactoring | Codex |
| Writing tests | Codex |
| Mechanical edits across many files | Codex |
| The same change repeated across multiple repos | Codex (one call per repo, in parallel) |
| Broad codebase research (map a subsystem, find all implementations of a pattern, trace data flow) | Codex (read-only) |
| Trivial single-edit changes (rename with known target, version bump, missing import, typo fix) | Claude |
| Quick lookups (a targeted grep or two, reading ≤2 known files) | Claude |
| READMEs, ADRs, design notes | Claude |
| Commit messages, PR descriptions | Claude |
| Architectural decisions | Claude |

A code subtask is treated as "trivial" only if the change is fully specified, fits in ≤2 files, needs no test/build loop, and Claude wouldn't have to read more than ~2 files to write the diff. When in doubt → Codex. The trivial shortcut is **off** entirely for tasks spanning more than one repo — N small edits across N repos are Codex work, one `--cd <repo>` call each.

## Inline-comment policy

Codex follows whatever it finds in your `CLAUDE.md` / `AGENTS.md` / `.codex/config.toml`. If nothing is specified, the default is **self-explanatory code, no inline comments** — Claude tells Codex this explicitly when it delegates.

## Output verbosity

Every `codex exec` call is launched with `-c model_verbosity=low`, and the prompt that Claude sends explicitly tells Codex to skip preamble/recap, to summarise the result in at most 5 short bullets, and to state explicitly when an acceptance criterion is unmet or the approach deviated — a partial result must never be presented as done. This is on for every call and overrides whatever `model_verbosity` is in your `~/.codex/config.toml` for the duration of the call. The reason is workflow-specific: Claude reviews each Codex run via `git diff`, so verbose Codex prose is pure token waste — the diff is the source of truth, not Codex' narration. There is no flag to opt out yet; if you actually need verbose Codex output, file an issue.

## Execution discipline

The prompt template also forbids Codex from running steps the orchestrator handles itself: no `git status` / `git diff` / `git log` (the orchestrator reviews via diff anyway), no formatter runs (`gofmt`, `prettier`, `black`, etc. — the orchestrator runs them after review), and tests at most once instead of after every fix. Codex is also told to stop opening additional files when the relevant excerpts are already embedded in the prompt. This stops Codex from repeatedly pulling its own diff and test output back into its own context, which is the dominant token leak in long-running Codex calls.

## Research delegation (no Claude subagents)

Broad codebase exploration — mapping how a subsystem works, finding every implementation of a pattern, tracing data flow — is delegated to Codex as a **read-only research run** (`codex exec --sandbox read-only`). Claude never spawns its own subagents in claudex mode: they run on Claude tokens, which is exactly the cost this plugin exists to avoid. Quick lookups (a targeted grep or two) Claude does inline; everything broader goes to Codex.

Research runs come in two forms, and the prompt is shaped accordingly:

- **Enumeration** — closed questions ("find every X", "list all callers of Y"). Claude runs one quick grep first and passes the hits as seed `path:line` candidates, explicitly framed as an incomplete list: Codex' job is to verify and extend it, including naming variants and indirection the grep would miss.
- **Investigation** — open questions ("how does X work", "why does Y happen", "where does this value actually come from"). These get **no seeds and no hypothesis**: Claude states the question neutrally and lets Codex work out its own answer — pre-chewed leads would anchor Codex on Claude's guess and turn research into rubber-stamping. If Claude merely wants a suspicion confirmed, that's not a research run at all; it verifies inline instead.

Questions that mix both forms are split into one research run per form, chained when one part feeds the other. Both forms share the rest of the prompt: what the answer will be used for, a compact repo-structure snapshot with explicit ignore-scopes, and a strict findings format — `path:line — one-line answer`, an "Open questions" section, plus a mandatory "Contradicts expectations" section so findings that disprove the question's premise get reported instead of reconciled away. On follow-up research in the same task, earlier findings are passed along labelled as unverified — to build on *and* to challenge. Claude spot-checks cited locations before embedding findings into implementation prompts.

Read-only research needs no project-trust entry and never uses the sandbox bypass. Enumeration always runs at your configured default reasoning effort; an investigation that needs cross-module causal reasoning may be escalated to `high` like any complex subtask (`--effort` still overrides everything).

## Sandbox & trust

`/claudex` invokes `codex exec --sandbox workspace-write` wherever the sandbox can actually work. How that plays out depends on the platform:

**macOS / Linux.** Codex only honours `workspace-write` when the project is in your trust list at `~/.codex/config.toml` (a `[projects."<path>"] trust_level = "trusted"` entry). When it isn't, Codex rejects every patch with `patch rejected: writing is blocked by read-only sandbox`. The runtime `-c projects.X.trust_level=...` override has been observed not to work reliably, so the plugin handles this in two places instead:

- **`/claudex:setup [dir]`** detects whether the target project (the optional directory argument, or the current one) is in the trust list and offers to add it (with explicit consent — your config file is not modified silently). Once trusted, `/claudex` runs at full sandbox without any bypass flag. Trust is per repo — before a multi-repo task, run it once per target repo.
- **At runtime**, if a `codex exec` call returns the exact `patch rejected: writing is blocked by read-only sandbox` error, `/claudex` retries that call once with `--dangerously-bypass-approvals-and-sandbox` and surfaces a notice in the final report. The retry only fires on that specific error string — other failures are surfaced as-is. The bypass flag is documented on `codex exec --help` for Codex CLI ≥ 0.128 and is the current replacement for the old `--full-auto` preset (which never existed on `codex exec`).

**Native Windows.** On Codex CLI 0.128, `--sandbox workspace-write` always degrades to read-only (the session header reports `sandbox: read-only`) and every patch is rejected — **trust-list entries do not change this**, in any TOML path form. A sandboxed first attempt would be a guaranteed wasted round-trip, so `/claudex` detects native Windows and adds `--dangerously-bypass-approvals-and-sandbox` to every write-capable Codex call from the start. Each report carries a notice that the call ran without sandboxing — this is the expected steady state on Windows, not an error, and `/claudex:setup` accordingly skips the trust-list flow there. If a future Codex version makes the sandbox work on Windows, this special-casing should be removed.

## Reasoning effort

By default, Codex runs at whatever level your `~/.codex/config.toml` configures (Codex' own fallback is `medium`). Codex supports `low`, `medium`, `high`, and `xhigh`, in that order.

When you don't pass `--effort`, Claude classifies each Codex subtask and adapts:

- **Complex subtasks** — algorithm design, ambiguous-spec refactors, multi-file design changes — get escalated to `high` for that one invocation.
- If your config default is **already `high` or `xhigh`**, Claude omits the override. Adding `high` against an `xhigh` default would actually lower reasoning, which is never what auto-escalation should do.
- **Standard subtasks** run at your configured default; no override is passed.
- Claude **never auto-lowers reasoning** and **never auto-escalates to `xhigh`**. Misjudging an easy task is a silent quality hit, so the floor stays at your configured default; jumping straight to `xhigh` is reserved for explicit user intent.

When you do pass `--effort <low|medium|high|xhigh>`, that level is forced for every Codex subtask in the call — including `low` and `xhigh`, which are only ever set when you explicitly ask for them.

## Parallelism rules

- File-disjoint Codex subtasks run **in parallel** (Claude launches them as background bash jobs while it works on docs).
- Subtasks that share any file run **sequentially**.
- When in doubt, sequential.

If Claude misjudges disjointness and two parallel jobs touch the same file, the review step flags it and recommends a revert.

## Commands

- `/claudex <task>` — orchestrate + delegate + review.
  - `--model <name>` — override Codex' default model for this call.
  - `--effort <low|medium|high|xhigh>` — force a Codex reasoning level for this call (bypasses auto-classification).
- `/claudex:setup [dir]` — verify Codex CLI is installed and authenticated; with `[dir]`, check/trust that directory instead of the current one (useful before multi-repo tasks).

## Not in this version

These are tracked as future work:

- [ ] Iteration loop (Claude reviews → re-delegates fixes to Codex automatically)
- [ ] Resume of Codex sessions
- [ ] Background mode with `/claudex:status` and `/claudex:result`

## License

MIT — see [LICENSE](LICENSE).
