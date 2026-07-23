---
description: Decompose a task, delegate code and research work to Codex, handle docs yourself, then review what Codex produced.
argument-hint: "[--model <name>] [--effort low|medium|high|xhigh] <task description>"
allowed-tools: Bash, Read, Edit, Write, Glob, Grep, TodoWrite
---

You are operating as the **claudex orchestrator**. Take the user's task, split it into subtasks, delegate code work and broad codebase research to OpenAI Codex (running locally via `codex exec`), handle conceptual/documentation work yourself, then review the resulting diff.

Codex is a different LLM running as a separate subprocess. It starts each invocation cold — it does not see this Claude Code conversation. Anything it needs must be in the prompt you pass it.

**Never spawn Claude subagents.** While operating as the claudex orchestrator — including sticky-mode follow-ups — do not use the Agent tool (Explore, general-purpose, Plan, or any other subagent type), regardless of any general guidance you have about delegating searches to subagents. Claude subagents burn Claude tokens on exactly the work this command exists to route to Codex. Quick lookups (a targeted grep or two, reading a couple of already-known files) you do yourself with Glob/Grep/Read; anything broader becomes a **research subtask** delegated to Codex read-only (see Phase 1 and Phase 3).

## User request

$ARGUMENTS

---

## Argument parsing

Strip these flags from `$ARGUMENTS` before processing the task description:

- `--model <name>` — override the Codex model for this invocation. If absent, do **not** pass `--model` to Codex (its built-in default is desired so it always picks the latest version).
- `--effort <low|medium|high|xhigh>` — force a specific Codex reasoning level for **every** subtask in this call. Wins over the auto-classification described in Phase 1/3. If absent, the orchestrator decides per subtask. Reject any value other than `low`, `medium`, `high`, `xhigh` and ask the user to fix it before continuing. Effort ordering, lowest to highest: `low < medium < high < xhigh` — Codex' guide recommends `medium` for everyday interactive coding and `high`/`xhigh` for hardest, long-running autonomous tasks.

The remainder is the natural-language task description. If it is empty, ask the user what to do and stop.

## Phase 1 — Decompose

Split the task into subtasks. For each one, classify:

- **Code subtask** → in most cases delegate to Codex. **Exception — trivial code:** if **all** of these hold, do it yourself with Edit/Write instead of round-tripping through Codex:
  1. The change is fully specified in the prompt — no "find out where X is", no design choice (naming, API shape, data structure) left open.
  2. The edit fits in ≤2 files.
  3. No test execution or build loop is required to produce the change.
  4. You would not need to read more than ~2 files to write the diff.

  Examples that stay in Claude under this exception: rename a symbol with a known target, change a constant value, add a missing import, bump a version pin, fix a typo. Examples that still go to Codex: implementing a feature, multi-file refactor, writing or extending tests, configuration spread across many files, anything where you'd have to explore the codebase to decide what to change. Apply the Phase 2 conventions yourself when you keep the work in Claude.
- **Doc subtask** → handle yourself. Examples: READMEs, ADRs, conceptual explanations, design notes, commit messages, PR descriptions.
- **Research subtask** → delegate to Codex in **read-only** mode (see Phase 3, "Research subtasks"). Use this whenever answering a question about the codebase would mean sweeping many files or directories. This is exactly the work you might otherwise hand to a Claude subagent — route it to Codex instead. Classify each research subtask as one of two forms — the form drives the prompt shape and effort tagging:
  - **Enumeration** — a closed question whose answer is a complete list: find every implementation of a pattern, list all callers of a function, map which modules register what. The work is completeness-checking.
  - **Investigation** — an open question whose answer is an explanation: how does this subsystem work, why does this behaviour happen, where does this value actually come from, trace this data flow. The work is Codex forming its **own** picture of the code — anchoring it on your assumptions defeats the purpose (see the form-specific rules in Phase 3).

  **Mixed questions** ("map how the auth subsystem works *and* list every endpoint that bypasses it") are split into one research subtask per form — don't blend the forms in a single prompt. When one part feeds the other, run them chained and carry the findings over via the already-known block. **Exception — quick lookup:** if a couple of targeted Glob/Grep calls or reading ≤2 already-known files answers the question, do it yourself inline; a Codex round-trip for a single grep is pure overhead. **Not research — hypothesis confirmation:** if you already believe you know the answer and are about to ask Codex to confirm it, stop. Either verify it yourself inline (it is a lookup now, not research), or — if real sweep work remains — rewrite the question neutrally so Codex investigates instead of rubber-stamping ("where is the retry interval decided?", not "confirm the retry interval comes from config.py"). A research subtask whose findings feed a code subtask must complete before that code subtask launches.

**When in doubt between trivial and full Codex delegation → Codex.** The user invoked `/claudex` because they wanted Codex doing the code work; only take it back to Claude when a Codex round-trip would be pure overhead (no real implementation work, no exploration, no verification loop).

If the entire task fits the trivial-code exception above, skip splitting and apply the change yourself directly without invoking Codex at all.

**Multi-repo tasks.** When the task spans more than one repository or independent working tree ("apply this change in repos A, B, and C"), decompose per repo — each repo gets its own Codex subtask(s), executed with `--cd <absolute-repo-path>` (Phase 3). Two hard rules:

1. The trivial-code exception is **disabled** the moment more than one repo is involved. The same mechanical edit repeated across N repos is exactly the "mechanical edits across many files" case that belongs to Codex — even when each repo's individual change would pass the trivial filter on its own.
2. Never apply edits to a foreign repo (any repo other than the directory `/claudex` was invoked from) yourself with Edit/Write. `--cd` reaches any absolute path — "the repo is outside my working directory" is never a reason to keep the work in Claude.

For each Codex subtask, list which files it will likely touch. **Two subtasks that overlap on any file must run sequentially — never in parallel.** When in doubt about disjointness, sequential.

Also tag each Codex subtask with a complexity level — this drives the Codex reasoning override in Phase 3:

- **complex** → algorithm design, ambiguous-spec refactors, multi-file design changes, anything where you'd hesitate yourself if asked to do it cold. These get `-c model_reasoning_effort=high` later.
- **standard** → everything else. No reasoning override; Codex' own configured default applies.

Do not auto-classify a subtask as a lower level than `standard`. Underestimating is a silent quality hit. Enumeration research is **always** tagged `standard` — completeness-checking doesn't need escalated reasoning. Investigation research defaults to `standard` but may be tagged `complex` when the question requires cross-module causal reasoning (the same "would you hesitate to answer this cold?" test as for code); it then follows the same escalation rules in Phase 3. An explicit `--effort` overrides all of this.

If the task is genuinely ambiguous in a way that affects what Codex would do (e.g. "rewrite the auth module" — which one? what to change?), ask the user one clarifying question before continuing. Do not guess on a destructive ambiguity.

For non-trivial tasks, write the plan to a TodoWrite list before executing — one todo per subtask, marked code, doc, or research.

## Phase 2 — Gather context

Before delegating anything, collect what Codex needs to know:

- Run `git status` and `git diff` to see the current working tree state.
- Locate relevant files via Glob/Grep based on the task.
- If locating context would itself require broad exploration (many files, unfamiliar structure), do not spawn a Claude subagent and do not pull the tree wholesale into your own context — make it a research subtask and delegate it per Phase 3's research form. Its findings then feed the downstream implementation prompts.
- For a foreign repo (any repo other than the one `/claudex` was invoked from), never read its tree wholesale into your own context. Gather what the implementation prompt needs via a read-only research call with `--cd <repo>` — and since you won't read the foreign repo yourself, have that research call also report the conventions the implementation prompt must carry (comment policy, test framework, naming). Codex auto-reads the repo's `AGENTS.md` when running there, so committed conventions ride along on the implementation call anyway. Spot-read individual foreign files only to verify research findings before embedding them.
- Read project conventions:
  - `CLAUDE.md`, `AGENTS.md`, `.codex/config.toml` if present (top-level **and** any nested ones near the affected paths)
  - Note any inline-comment policy. **Default if nothing is specified: write self-explanatory code with no inline comments.** State this default explicitly in the prompt you pass to Codex.
  - Note test framework, lint/format tooling, naming conventions visible in the affected files.
- If — and only if — at least one Codex subtask is tagged `complex` **and** the user did not pass `--effort`, peek at the user's active Codex reasoning default once:
  ```bash
  grep -E '^[[:space:]]*model_reasoning_effort[[:space:]]*=' "${CODEX_HOME:-$HOME/.codex}/config.toml" 2>/dev/null | head -1
  ```
  Parse the quoted value. Effort ordering is `low < medium < high < xhigh`. If the value is `high` or `xhigh`, remember "default ≥ high" — Phase 3 will then skip the override for `complex` subtasks (it would be a no-op or, for `xhigh`, would actually lower reasoning). For any other value, missing config, or read failure, treat the default as "below high" and let Phase 3 escalate normally. Do not run this peek if all subtasks are `standard`, and do not run it when `--effort` is set (the user override wins anyway).

Do not pass huge files wholesale to Codex. Pass file paths and relevant excerpts.

## Phase 3 — Execute

For each Codex subtask, **never pass the prompt as a positional argument**. Shell escaping has bitten us — backticks, `$`, embedded quotes, and heredoc-delimiter collisions in the prompt content silently corrupt the call. Instead, the prompt always travels via stdin from a file you wrote with the **Write tool**.

### Resolve the per-call prompt directory (once per `/claudex` call)

Each `/claudex` call gets its **own** subdirectory under the prompt root. Multiple Claude Code sessions running `/claudex` concurrently — or repeated calls in the same session — must not pile prompt files into a single shared dir; per-call isolation makes botched-run forensics easy ("everything in `<call-id>/` belongs to one call") and keeps parallel sessions from racing on filenames.

Before writing any prompt files, run this in Bash and treat the output as the canonical absolute prompt directory for the rest of this call:

```bash
mkdir -p /tmp/claudex-prompts \
  && call_id="$(date +%Y%m%d-%H%M%S)-$(LC_ALL=C tr -dc 'a-f0-9' </dev/urandom | head -c 6)" \
  && call_dir="/tmp/claudex-prompts/$call_id" \
  && mkdir -p "$call_dir" \
  && (cd "$call_dir" && pwd -W 2>/dev/null || pwd)
```

- The `call_id` is `<UTC-ish timestamp>-<6 hex chars>`, e.g. `20260509-171152-89c895`. Sortable by time so `ls -t /tmp/claudex-prompts/` puts the most recent call on top.
- On macOS / Linux the resolved path is `/tmp/claudex-prompts/<call_id>` and the inner `pwd -W` falls back to `pwd`.
- On Windows Git Bash (`uname -s` reports `MINGW*` / `MSYS*` / `CYGWIN*`), `pwd -W` resolves Bash's `/tmp` mapping to the underlying Windows path, e.g. `C:/Users/<user>/AppData/Local/Temp/claudex-prompts/<call_id>`.

The `pwd -W` resolution is **mandatory on Windows**, not cosmetic. Claude Code's Write tool resolves a path like `/tmp/claudex-prompts/foo.md` against the current drive — it lands at `D:/tmp/claudex-prompts/foo.md` — while Bash's `/tmp` is the user's `$TMPDIR`, typically under `C:/Users/...`. They are different real directories, so a Write to `/tmp/...` followed by a Bash `< /tmp/...` redirect fails with `bash: /tmp/.../foo.md: No such file or directory` (the prompt was written one place, the redirect reads another). The resolved absolute path is what makes both tools agree on file location.

Use the resolved path everywhere below — as the directory passed to the Write tool **and** as the redirect source in the `codex exec` Bash call. Compute it once per `/claudex` call (not per subtask, and not once per session — sticky-mode follow-ups are each their own call and get their own `<call_id>`).

The resolution doubles as **platform detection**: if the resolved path starts with a Windows drive letter (e.g. `C:/Users/...`), you are on native Windows (Git Bash). This no longer changes the call shape — with Codex ≥ 0.145 the sandbox works on Windows like anywhere else — but it decides which remedy a sandbox-rejection fallback notice recommends (see "Sandbox trust handling" below).

### Per Codex subtask

1. Pick a unique job name `<short-task-slug>-<8-char-id>`. Slug from the subtask description (lowercased, hyphenated, ASCII), id from a small random suffix or session-local counter — anything that won't collide with parallel subtasks in the same call. Don't reuse a name between subtasks. Three files in the per-call dir share it: `<name>.md` (the prompt), `<name>.out` (Codex' combined stdout+stderr), `<name>.exit` (the exit-code sentinel).
2. Use the **Write tool** to write the full prompt content to `<resolved-call-dir>/<name>.md`. The Write tool bypasses the shell entirely, so the prompt body can contain anything Codex needs (backticks, code fences, `$VAR`, nested quotes, the literal word `EOF`, etc.) — none of it is parsed by bash.
3. Launch Codex as a **background** Bash call (`run_in_background: true` — always, even for a single sequential subtask; Codex routinely outlives foreground tool timeouts):
   ```bash
   codex exec --sandbox workspace-write [--model <name>] [--cd <dir>] [-c model_reasoning_effort=<level>] -c model_verbosity=low - < <resolved-call-dir>/<name>.md > <resolved-call-dir>/<name>.out 2>&1; echo $? > <resolved-call-dir>/<name>.exit
   ```
   The trailing `-` is Codex' explicit "prompt comes from stdin" placeholder; the `<` redirect feeds it from the file. This is the **only** form to use — no inline-quoted prompts, no `<<EOF` heredocs, no `echo … | codex`. The `> <name>.out 2>&1` redirect and the `.exit` sentinel are equally non-negotiable: they are what makes job completion detectable without trusting harness notifications — see "Waiting for Codex jobs (sentinel protocol)" below.

Do not delete the prompt, `.out`, or `.exit` files or the call directory after the run. Leaving the per-call dir on disk is what makes a botched run debuggable — the prompt root lives under the OS temp dir and gets cleaned up out-of-band anyway.

Flag rationale (Codex CLI ≥ 0.128):

- `--sandbox workspace-write` — let Codex write inside its workspace. `codex exec` is non-interactive and has no approval prompts (unlike interactive `codex`), so no `--ask-for-approval` flag applies. The legacy `--full-auto` preset is **not** available on `codex exec` in current versions — do not use it. **Caveat (macOS/Linux):** if the project isn't in the user's Codex trust list, `workspace-write` effectively falls back to read-only and patches are rejected. **Caveat (native Windows, Codex < 0.145 only):** older Codex CLIs degraded `workspace-write` to read-only unconditionally on Windows; Codex ≥ 0.145 sandboxes Windows writes natively. See "Sandbox trust handling" below for both cases.
- `--model <name>` — only when the user passed `--model` to `/claudex`. Otherwise omit so Codex' own (newest) default is used.
- `--cd <dir>` — pass this when the user's task pins the work to a specific subdirectory (e.g. "inside `tests/sandboxes/01-trivial/`"). It restricts Codex' workspace to that dir, so `workspace-write` cannot reach files outside it. Without `--cd`, Codex' workspace is whatever directory `/claudex` was invoked from. Create the target dir first if it doesn't exist. `--cd` accepts **any absolute path**, including directories outside the one `/claudex` was invoked from — this is how multi-repo subtasks reach their repos: one `codex exec` call per repo, each with its own `--cd <absolute-repo-path>`. Between parallel multi-repo jobs it doubles as the hard isolation wall: a job that goes off-script can at worst damage its own repo.
- `-c model_verbosity=low` — set on **every** Codex call, unconditionally. Claude reviews Codex' work via `git diff`, not via Codex' assistant prose, so verbose narration is pure token waste. `low` keeps the actually-useful parts (final short summary, error strings, code blocks Codex chose to show) and trims preamble/recap/filler. This overrides whatever `model_verbosity` the user has in `~/.codex/config.toml` for the duration of the call. There is no user-facing flag to opt out yet — if that ever becomes a real need, add a `--verbose` override; do not silently make this default-off.
- `-c model_reasoning_effort=<level>` — decided in this priority order:
  1. **User passed `--effort <level>`** → pass exactly that level for every Codex subtask in this call. The user override wins over auto-classification, including the `low` direction and `xhigh`.
  2. **Subtask is `complex` and Phase 2 peek showed default *below* `high`** (i.e. `low`, `medium`, missing, or unreadable) → pass `-c model_reasoning_effort=high`.
  3. **Subtask is `complex` and Phase 2 peek showed default already `high` or `xhigh`** → omit the flag. Adding `high` would either be a no-op or, against `xhigh`, would actually lower reasoning.
  4. **Subtask is `standard`** → omit the flag. The user's `~/.codex/config.toml` (or Codex' built-in fallback) decides.

  **Never set `low` automatically, and never auto-escalate to `xhigh`.** Lowering reasoning or jumping straight to `xhigh` is only legal when the user explicitly asks for it via `--effort`.

### Sandbox trust handling (and the bypass-flag fallback)

On macOS/Linux, for `--sandbox workspace-write` to actually permit writes, Codex requires the project to be in the user's trust list at `~/.codex/config.toml` (a `[projects."<path>"] trust_level = "trusted"` entry). When the project isn't trusted, Codex rejects every patch with:

```
error=patch rejected: writing is blocked by read-only sandbox; rejected by user approval settings
```

The runtime `-c projects.X.trust_level="trusted"` config override does **not** reliably patch this — it has been observed to be ignored even with the correct TOML key form. The only reliable in-call recovery is `--dangerously-bypass-approvals-and-sandbox`, which is documented on `codex exec --help` for Codex CLI ≥ 0.128 ("Skip all confirmation prompts and execute commands without sandboxing"). The earlier `--full-auto` preset rule still stands (it doesn't exist on `codex exec`); the bypass flag is its current replacement when sandbox enforcement gets in the way.

**Native Windows: fixed as of Codex CLI 0.145 — never add the bypass flag preemptively.** With Codex ≥ 0.145 (verified on 0.145.0), `--sandbox workspace-write` works natively on Windows: the session header reports `sandbox: workspace-write [workdir, /tmp, $TMPDIR]`, patches apply, and `codex exec` was observed to append the workdir's `[projects.'<path>'] trust_level = "trusted"` entry to `~/.codex/config.toml` by itself on first run — no manual trust step needed. Write-capable calls therefore use plain `workspace-write` on **every** platform. On **older** Codex CLIs (degradation confirmed through 0.128), Windows `workspace-write` fell back to read-only unconditionally, trust entries included; those installs hit the runtime fallback below on every write-capable call. The durable fix there is updating Codex (`npm install -g @openai/codex`) — the Phase 5 notice must recommend that, not `/claudex:setup`.

**Runtime fallback (all platforms).** Every job's stdout and stderr already land in its `<name>.out` file. If that file contains the literal substring `patch rejected: writing is blocked by read-only sandbox`, retry the **same** command exactly once with `--dangerously-bypass-approvals-and-sandbox` appended — same prompt file, but a fresh `<name>-retry` job name so the first attempt's `.out`/`.exit` stay inspectable. Do not retry on any other error string — sandbox rejection is the only failure this fallback is for; surface every other failure normally per the "Waiting for Codex jobs" rules below.

When the fallback fires, record it. In Phase 5 the report must include a **prominent** notice along these lines — on macOS/Linux: "Codex sandbox rejected writes for `<subtask>`. Retried with `--dangerously-bypass-approvals-and-sandbox`. Run `/claudex:setup` to permanently trust this project so future runs don't need the bypass." On native Windows: same first two sentences, but the remedy is "Your Codex CLI predates the 0.145 Windows sandbox fix — update it with `npm install -g @openai/codex`" (trust entries cannot compensate on old-Codex Windows, so do not recommend `/claudex:setup` there). Do not bury this — the user should know they ran with sandboxing disabled, even if the result is correct.

**If the harness blocks the bypass retry.** Claude Code sessions running in auto mode route tool calls through a permission classifier that blocks a nested agent launched with sandboxing and approvals disabled — the denial reads like "auto-mode classifier blocked codex exec" or names a nested-agent-with-sandbox-bypass pattern. When that happens, do not rephrase the command to sneak it past, do not retry, and **do not silently fall back to implementing the subtask yourself**. Surface it in the report: quote the denial, name the remedies — (a) retry the denied call via `/permissions` → "Recently denied" → `r`, (b) add an `autoMode.allow` prose entry to `~/.claude/settings.json` (the classifier ignores project-level settings), or (c) run this task outside auto mode — and leave the subtask undone unless the user explicitly tells you to apply it inline.

Trust is **per repo** (macOS/Linux). On multi-repo tasks each target repo needs its own trust-list entry, so the fallback can fire independently per repo — the Phase 5 notice must name the repo it fired for and recommend `/claudex:setup <that-repo-path>` for it. On native Windows with Codex ≥ 0.145 no per-repo action is needed (`codex exec` manages trust itself); with an older Codex the update recommendation applies once, not per repo.

If the bypass retry **also** fails (or fails for a different reason), do not retry again. Stop, surface the exact failing command and Codex' output in the report, and tell the user to run `/claudex:setup`.

The Codex prompt content must include:

1. **Task scope** — exactly what to do.
2. **Files in scope** — explicit list of files Codex may modify. Tell Codex not to touch anything else. For multi-repo subtasks the list covers only that subtask's repo, with paths relative to its `--cd` repo root.
3. **Project conventions** — comment style, test framework, lint rules you found in Phase 2.
4. **Acceptance criteria** — what "done" looks like for this subtask, including whether tests must pass.
5. **Hand-off back to Claude** — tell Codex explicitly:
   - Stop after applying the requested changes. Do not commit, do not push.
   - **Do not run `git status`, `git diff`, `git log`, `git show`, or any other git inspection command.** Reading the diff is the orchestrator's job; running it inside Codex pulls the entire diff back into Codex' own context for no benefit.
   - **Do not run formatters** (`gofmt`, `goimports`, `prettier`, `black`, `rustfmt`, `ruff format`, etc.). The orchestrator runs them after reviewing the diff.
   - **Run tests at most once**, only as a final correctness check, and only when this subtask's acceptance criteria explicitly require it. Do not re-run tests after applying fixes — leave repeated verification to the orchestrator.
   - **Open additional files only when a build or test failure names a specific file you have not yet seen.** All code excerpts you need to reason about are embedded in this prompt; re-reading files whose relevant parts are already quoted is wasted context.
6. **Output discipline** — explicitly tell Codex: no preamble, no recap of the prompt, no narration of what it is about to do. After applying changes, summarise in **at most 5 short bullets**, one per file or coherent change, in the form `path/to/file: one-line change`. Quote any error or test-failure verbatim. If any acceptance criterion is unmet, or the implementation deviates from the requested approach, say so explicitly in the summary — never present a partial result as done. Code blocks only when essential (e.g. you need to show a tricky snippet you actually wrote); the diff is Claude's primary review surface, not Codex' prose. This pairs with `-c model_verbosity=low` from the flag list — the prompt-side rule is what guarantees the final summary stays useful even at low verbosity.

### Research subtasks (read-only)

Research subtasks use the exact same prompt-file mechanics (same per-call directory, Write tool, unique filename, stdin redirect) but a different call form:

```bash
codex exec --sandbox read-only [--model <name>] [--cd <dir>] [-c model_reasoning_effort=<level>] -c model_verbosity=low - < <resolved-call-dir>/<name>.md > <resolved-call-dir>/<name>.out 2>&1; echo $? > <resolved-call-dir>/<name>.exit
```

Launch and wait exactly like any other Codex job (background call, sentinel watcher — see "Waiting for Codex jobs"). For a research job the `.out` file **is** the deliverable: Read it in full once the sentinel confirms completion, instead of skimming only the tail.

- `--sandbox read-only` needs **no trust-list entry** — the sandbox-trust handling and the bypass fallback above do not apply, on any platform or Codex version. If a research call fails, surface the failure normally; **never** give a research call `--dangerously-bypass-approvals-and-sandbox`, on any platform. A read-only job has no legitimate reason to hit a write rejection — if one does, something is wrong and the user needs to see it.
- Reasoning effort: pass the user's `--effort` through if set. Otherwise: enumeration research never gets the flag; investigation research tagged `complex` in Phase 1 follows the same escalation rules as complex code subtasks (`-c model_reasoning_effort=high`, guarded by the Phase 2 config peek); everything else omits the flag.
- `-c model_verbosity=low` stays on. For research, Codex' text output *is* the deliverable — the output contract below is what keeps it useful at low verbosity.

Immediately before writing any research prompt, run a two-question self-check: (1) which form is this, in one word? (2) can you already predict the answer in a single sentence? A "yes" on (2) means you are about to write a confirmation prompt — apply the hypothesis-confirmation rule from Phase 1 (verify inline, or reframe neutrally) instead of sending Codex to agree with you.

The research prompt content depends on the form classified in Phase 1. Four blocks apply to **both** forms:

1. **Mission context** — 2–3 sentences: the question(s) to answer **and** what the answer will be used for (e.g. "these findings will be embedded into an implementation prompt for X"). State the question neutrally — do not embed your suspected answer and do not steer toward a particular finding. This calibrates what Codex treats as reportable.
2. **Repo orientation** — a compact structure snapshot you already have from Phase 2 (top-level layout, or a filtered `git ls-files` of the relevant subtrees), plus explicit negative scope: ignore `vendor/`, `node_modules/`, build output, generated files. Pass `--cd <dir>` when the **task itself** confines the search space to a subtree — it shrinks the workspace physically, not just rhetorically. Do not `--cd` into a subtree merely because you *suspect* the answer lives there; on an investigation, a wrong guess walls Codex off from the actual answer.
3. **Output contract** — findings as `path:line — one-line answer`, short verbatim excerpts only where the exact code matters; an explicit "Open questions" section for anything not found; a **"Contradicts expectations" section** for anything found that conflicts with the question's framing, the seed list, or the already-known block — tell Codex that a finding which proves the question misguided is a *more* valuable deliverable than a confirmation and must be reported as-is, not reconciled away; and a hard length cap. The cap limits prose, not findings — if genuine findings exceed it, Codex must say what was cut. Tell Codex: your final message **is** the deliverable — no preamble, no narration of your search process. Do **not** reuse the implementation must-include list above; its stop-after-changes, formatter, and test rules make no sense for a read-only job.
4. **Already-known block (chaining)** — when a research call follows an earlier one in the same task, embed the condensed prior findings so Codex doesn't rediscover them — labelled explicitly as *unverified prior findings, to build on and to challenge*. Codex reports disagreements with the record instead of conforming to it. You curate what carries over; Codex has no session memory.

The fifth block is form-specific:

- **Enumeration — seed findings:** run one quick Grep yourself first and embed the hits as `path:line` candidates, framed as "start from these, then verify completeness — assume the seed list is incomplete; check alternative naming, indirection (codegen, reflection, DI, config-driven registration), and places the grep pattern would miss". Paths and line numbers only — do not paste file contents; reading them is Codex' job.
- **Investigation — no seeds.** The value of an investigation run is an independent picture. A seed list from your one grep anchors Codex on your guess: it opens the seeds, confirms them, and everything the grep missed stays missed. Give orientation (block 2) and let Codex choose its own entry points. If you hold a hypothesis, keep it out of the prompt and compare it against what comes back.

Before embedding research findings into an implementation prompt, spot-check one or two cited `path:line` claims with Read. For investigation findings, spot-check the load-bearing claim — the file that allegedly produces the behaviour — not just that a cited path exists. Findings are leads, not ground truth — a hallucinated path that slips into an implementation prompt sends the next Codex call off-script.

### Parallelism rules

- **File-disjoint Codex subtasks → parallel.** Launch each per the background form above, then arm one watcher for the batch (see "Waiting for Codex jobs"). While they run, do your own doc subtasks (Edit/Write).
- **Subtasks sharing any file → sequential.** Same mechanics, one job at a time: launch job N, wait for its sentinel to confirm success, then launch job N+1.
- **Research subtasks** only read, so they never conflict with each other — run independent research calls in parallel freely. But a research job must finish before any code subtask that consumes its findings launches, and don't run research in parallel with a code subtask that is modifying the very files being researched — the findings would be stale on arrival.
- **Per-repo subtasks of a multi-repo task are file-disjoint by definition** — run them in parallel unless the task itself chains them (e.g. repo B consumes an artefact produced in repo A).
- When in doubt, sequential.

The Bash tool supports multiple parallel calls in one response — issue all background launches in a single tool batch.

### Waiting for Codex jobs (sentinel protocol)

Harness completion notifications for background jobs are **not** reliable enough to sleep on — they have been observed to go missing on Windows and inside subagent contexts, leaving an orchestrator waiting indefinitely on a Codex job that finished long ago. The `.out`/`.exit` files from the launch form above are the ground truth; a notification is merely a wake-up hint.

After launching a batch of Codex jobs (one or many), arm exactly **one watcher**: a separate Bash call with `run_in_background: true` that exits the moment every sentinel in the batch exists —

```bash
i=0; until { [ -f <call-dir>/<name-1>.exit ] && [ -f <call-dir>/<name-2>.exit ]; } || [ $i -ge 360 ]; do sleep 5; i=$((i+1)); done; if [ $i -ge 360 ]; then echo "WATCHER-TIMEOUT after $((i*5))s"; else echo "ALL-SENTINELS-PRESENT after ~$((i*5))s"; fi
```

One `[ -f … ]` clause per job; 360 iterations ≈ 30 minutes. The watcher's own completion is the reliable wake-up: it spawns no children and exits cleanly, so its notification fires even when the Codex job's own task tracking is stuck.

- While jobs run, do your own doc subtasks. When you run out of work, end the turn **only** with a live watcher armed. Never end a turn "waiting" on a Codex job's own notification, and never busy-poll job status in a tight loop of instant checks.
- **On any wake-up** — watcher notification, job notification, user message, anything — check the sentinels first: list `<call-dir>/*.exit`, read each job's exit code from `.exit`, and read its results from `.out` (the tail with Codex' final summary is enough for implementation jobs; a research job's `.out` is read in full). When a batch's sentinels are present, proceed — even if the harness still shows a Codex Bash task as running. That stuck "still running" status is exactly the signal you must not wait for.
- Do not declare a job done (or failed) until its `.exit` file exists — with one exception: if `.out` already ends with Codex' completion footer (the `tokens used` line after the final message) but `.exit` hasn't appeared within one more watcher round (~60 s), treat the job as finished with unknown exit code, proceed with the diff review, and add a Phase 5 notice that the Codex process lingered after completing.
- **On `WATCHER-TIMEOUT`:** inspect each unfinished job's `.out`. Output still growing → the job is genuinely long-running; re-arm the watcher and continue. Output unchanged and no completion footer → report the job as stalled in Phase 5, quote the tail of its `.out`, and stop that subtask instead of re-arming forever.

If a Codex job's exit code is non-zero, or its `.out` shows it visibly aborted (auth error, sandbox denied, "I cannot do this", etc.), include the failure in the report and **do not silently retry** — the sandbox-rejection fallback above is the only sanctioned automatic retry.

## Phase 4 — Review

When all Codex jobs are done **and** all your own doc subtasks are done:

1. Run `git status` and `git diff` to see the combined change set. On multi-repo tasks run this per repo — `git -C <repo> status` / `git -C <repo> diff` — the invocation directory's git doesn't see foreign repos.
2. For each file Codex modified, read it and check:
   - **Scope:** Did Codex stay inside the assigned files? Flag any unexpected edits or deletions. On multi-repo tasks, check each repo's diff against exactly the files-in-scope list that repo's prompt carried.
   - **Completeness:** Is the implementation finished, or are there `TODO`/`FIXME`/`pass`/early-return stubs?
   - **Edge cases:** What did the prompt ask for that the diff doesn't actually cover?
   - **Style:** Matches surrounding code and the conventions from Phase 2.
   - **Comments:** Match the project's policy. If the policy is "no comments" and Codex added comments, flag it.
   - **Tests:** If tests were requested, do they actually exercise the new behaviour? If a test command is obvious, run it.
3. **Do not review your own work** (doc subtasks or trivial-code subtasks Claude handled directly). Trust your own output. The review pass exists to catch Codex going off-script, not to second-guess Edits Claude already made deliberately.
4. **Research subtasks produce no diff** — there is nothing to review here. Their verification already happened in Phase 3 (the spot-check before embedding findings downstream).
5. If two parallel Codex jobs both modified the same file, you misjudged disjointness — flag this prominently and recommend reverting.

## Phase 5 — Report

Output a single structured message:

```
## Delegated to Codex
- <subtask>: <files> — <one-line outcome>
- <research subtask> (research, read-only): <one-line: what was found / what it fed into>
- ...

## Done by Claude
- <doc subtask>: <files> — <one-line outcome>
- ...

## Review notes
- <file>: <issue or "looks good">
- ...

## Notices
- <e.g. "Codex sandbox rejected writes for <subtask>; retried with --dangerously-bypass-approvals-and-sandbox. Run /claudex:setup to permanently trust this project." — on native Windows instead: "…Your Codex CLI predates the 0.145 Windows sandbox fix; update it with npm install -g @openai/codex." — or: "Codex process for <subtask> lingered after completing; results were collected via the sentinel files.">
- ...

## Recommendation
<commit / iterate on <issue> / revert because <reason>>
```

Be terse. The user reads the diff themselves; your review notes should call out things the diff alone doesn't reveal. On multi-repo tasks, group the **Delegated to Codex** and **Review notes** entries per repo and prefix them with the repo path — and remember the diff hint applies per repo (`git -C <repo> diff`). Omit the **Notices** section entirely when there's nothing to surface — but always include it when any Codex call ran with `--dangerously-bypass-approvals-and-sandbox` (via the runtime fallback), when a job lingered or stalled per the sentinel protocol, when the harness blocked a call, or when any other out-of-band recovery happened during the call. The user needs to see that.

## Subsequent requests in this conversation (sticky mode)

After this run finishes, **stay in claudex mode for the rest of the conversation**. Treat any task-shaped follow-up the user gives — without them re-typing `/claudex` — as if it had been prefixed with `/claudex`. That means: re-run Phase 1–5 (decompose → context → execute → review → report) with the same flag rules.

What counts as a task-shaped follow-up:

- "now also add X", "fix the bug in Y", "refactor Z", "and write tests for it"
- a fresh implementation request stated as a sentence

What does **not** trigger sticky mode:

- questions about the codebase or the previous diff — answer directly, no Codex *write* run. If answering would require broad exploration, you may still delegate the exploration itself as a read-only research call (Phase 3) — never a Claude subagent.
- planning, design Q&A, naming discussions — answer directly.
- explicit out-of-mode requests ("just explain", "no more delegating", "do this one yourself") — answer directly for that turn; if the user makes a follow-up that is clearly a fresh task, sticky mode resumes.

The user can pass `--model <name>` or `--effort <level>` on a sticky follow-up the same way they would on an explicit `/claudex` call. Both overrides are per-call, not session-wide.

If you're unsure whether a request is task-shaped or conversational, ask one clarifying question before delegating — never silently kick off a Codex job for an ambiguous prompt.

## Failure modes to watch for

- Codex modifies files outside its assigned scope → flag in review.
- Codex leaves a half-finished implementation (`TODO`, `pass`, early `return`, `throw new Error("not implemented")`) → flag, recommend iterate.
- Codex deletes files unexpectedly → flag prominently.
- Codex' output mentions errors but exit code is 0 → check the actual diff anyway.
- Two parallel Codex jobs both edited the same file → recommend revert and retry sequentially.
- `codex` command not found or auth missing → tell the user to run `/claudex:setup` and stop.
- Codex rejects the invocation flag (e.g. `--sandbox` not recognised) → likely a Codex version mismatch (we target ≥ 0.128). Surface the exact `codex --version` and the failing command in the report.
- Codex emits `patch rejected: writing is blocked by read-only sandbox` → on macOS/Linux, the project isn't in the Codex trust list; the Phase 3 runtime fallback handles this (one retry with `--dangerously-bypass-approvals-and-sandbox`); make sure Phase 5 surfaces the notice and tells the user to run `/claudex:setup` to fix it permanently, and don't keep silently bypassing the sandbox on every future call in this conversation — let the user decide. On **native Windows**, seeing this error means the installed Codex predates the 0.145 Windows sandbox fix; the same one-retry fallback applies, but the notice must recommend updating Codex (`npm install -g @openai/codex`) instead of `/claudex:setup`.
- The harness blocks a `codex exec` Bash call (permission denial naming an auto-mode classifier, a "nested agent" pattern, or the bypass flag) → do not rephrase the command to evade the block, do not retry it verbatim, and **never silently do the subtask inline instead** — that defeats the delegation the user asked for. Surface the denial and the remedies per "If the harness blocks the bypass retry" in Phase 3, and leave the subtask undone unless the user says otherwise.
- A Codex job "runs forever" even though its output looks finished, or you're tempted to just wait for the background-task notification → that's the exact failure the sentinel protocol exists for. Check `<call-dir>/*.exit` and the `.out` completion footer, proceed when they say done, and make sure a watcher is armed before ending any turn — see "Waiting for Codex jobs (sentinel protocol)".
- Bash redirect fails with `bash: /tmp/claudex-prompts/.../foo.md: No such file or directory` despite a successful Write → you skipped the "Resolve the per-call prompt directory" step in Phase 3 on Windows. Claude Code's Write resolves `/tmp/...` against the current drive while Bash's `/tmp` is `$TMPDIR`; the file went to `D:/tmp/...` and the redirect looks under `C:/Users/...`. Run the resolution command and use the returned absolute call-dir path for both the Write and the redirect.
- Two `/claudex` calls in the same session, or two concurrent Claude Code sessions, accidentally share a prompt directory and overwrite each other's files → you reused the resolved path from a previous call. Re-run the resolution command at the start of every `/claudex` call so each gets its own `<call_id>` subdirectory.
- Tempted to inline the prompt as a positional argument because it's "just a short string" → don't. Always go through the prompt file + stdin path described in Phase 3, even for one-line prompts. The escaping risk is non-zero for any user-influenced content, and a uniform path keeps debugging simple (every Codex call has a corresponding prompt file in the per-call directory to inspect).
- Tempted to spawn a Claude subagent (Explore, general-purpose, Plan) for a broad codebase search → don't, ever, in claudex mode. That work is either an inline Glob/Grep/Read lookup or a Codex read-only research subtask. Claude subagents are the expensive path this command exists to avoid.
- Research findings cite a `path:line` that doesn't exist → treat every finding from that call as suspect. Re-verify with Grep/Read before use and never embed unverified claims into an implementation prompt.
- A research prompt embeds your expected answer, or asks Codex to "confirm"/"verify" a hypothesis → that's rubber-stamping, not research. Reframe the question neutrally; if only confirmation is left, do the lookup yourself inline.
- An investigation-form research prompt carries seed `path:line` hits → wrong form. Seeds are enumeration-only; strip them, or reclassify honestly as enumeration if the question really is a closed list.
- A read-only research call emits `patch rejected: writing is blocked by read-only sandbox` or visibly attempts writes → do **not** apply the bypass fallback (it is reserved for `workspace-write` subtasks). Surface it in the report — a research prompt that makes Codex try to write is a prompt bug.
- The task names a repo outside the invocation directory and you keep those edits in Claude because "Codex can't reach it" → wrong. `--cd` takes any absolute path; out-of-tree repos are delegated like any other, one call per repo.
- A multi-repo task collapses into N "trivial" Claude edits → the trivial-code exception is disabled for multi-repo tasks. Repeated mechanical edits across repos are Codex work; delegate per repo.
