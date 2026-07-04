# 12 — Research delegation (read-only Codex, no Claude subagents)

## Goal

Verify the research-subtask path end to end:

1. **Routing.** A task whose first half requires broad codebase exploration produces a dedicated research subtask that runs as `codex exec --sandbox read-only …` — and the orchestrator **never** uses the Agent tool (no Explore / general-purpose / Plan subagents), neither for the research nor for anything else in the call.
2. **Prompt content.** The research prompt file under `/tmp/claudex-prompts/<call_id>/` contains the five mandatory blocks: mission context (what the findings feed), repo orientation with negative scope, seed `path:line` findings from an orchestrator-side grep, an output contract (`path:line — one-line answer`, "Open questions" section, length cap, "your output is the deliverable"), and — only on chained research — an already-known block.
3. **Sequencing.** The research call completes **before** the implementation subtask launches, and its findings are embedded in the implementation prompt.
4. **No bypass, ever.** The research call is never retried with `--dangerously-bypass-approvals-and-sandbox`, and it works without a project-trust entry.

If the orchestrator answers the exploration inline instead (a single grep would not be enough here — mapping 10 handler modules requires reading them), or spawns a Claude subagent, this is a regression.

## Setup

```bash
rm -rf tests/sandboxes/12-research
mkdir -p tests/sandboxes/12-research/handlers
cat > tests/sandboxes/12-research/core.py <<'EOF'
REGISTRY = {}

def register(name):
    def deco(fn):
        REGISTRY[name] = fn
        return fn
    return deco
EOF
for i in $(seq -w 1 10); do
  cat > "tests/sandboxes/12-research/handlers/handler_$i.py" <<EOF
import sys, os
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))
from core import register

@register("event_$i")
def handle_$i(payload):
    return {"event": "event_$i", "payload": payload}
EOF
done
```

## Invocation

```
/claudex First research how event handlers are registered across tests/sandboxes/12-research/ — which modules register which event names, and through what mechanism. Then, using those findings, create tests/sandboxes/12-research/manifest.py exposing EXPECTED_EVENTS (a sorted list of every registered event name) and a test test_manifest.py in the same directory that imports every handler module and asserts that the keys of core.REGISTRY equal EXPECTED_EVENTS. Verify with `python -m pytest tests/sandboxes/12-research/test_manifest.py`. Do not modify anything outside tests/sandboxes/12-research/.
```

## Expected behaviour

- Phase 1 produces (at least) two Codex subtasks: a **research** subtask (read-only) and a **code** subtask (workspace-write). The TodoWrite plan marks them accordingly.
- **No Agent tool calls anywhere in the transcript.** Any subagent spawn is a hard failure of this scenario.
- The research invocation looks roughly like:
  ```bash
  codex exec --sandbox read-only --cd tests/sandboxes/12-research -c model_verbosity=low - < /tmp/claudex-prompts/<call_id>/<research-slug>-<id>.md
  ```
  - `--sandbox read-only`, not `workspace-write`.
  - No `-c model_reasoning_effort=…` (research is always `standard`; only an explicit `--effort` would add it).
  - No `--dangerously-bypass-approvals-and-sandbox`, and no retry with it — even if the project is not in the Codex trust list (read-only doesn't need trust; that's part of what this scenario proves).
- The research **prompt file** contains, in substance (wording is up to the orchestrator):
  - mission context: the registration question *and* that the findings feed a manifest/test implementation;
  - repo orientation: the sandbox layout (`core.py`, `handlers/…`) and/or explicit scope limits;
  - seed findings: `path:line` hits from a grep the orchestrator ran itself (e.g. for `register(`) — paths and line numbers, not pasted file contents;
  - output contract: `path:line — one-line answer` format, an "Open questions" section, a length cap, and "your final message is the deliverable / no narration";
  - **not** the implementation must-include rules (no "stop after applying changes", no formatter/test rules — those make no sense read-only).
- The research job finishes before the implementation `codex exec` (with `--sandbox workspace-write`) launches, and the implementation prompt embeds the research findings (the event-name list and/or the registration mechanism).
- Before embedding, the orchestrator spot-checks at least one cited `path:line` with Read/Grep.

## Verify

```bash
# Implementation output exists and tests pass.
ls tests/sandboxes/12-research/
# expect additionally: manifest.py, test_manifest.py
python -m pytest tests/sandboxes/12-research/test_manifest.py

# Two prompt files in the per-call directory: one research, one implementation.
prompt_root="$(cd /tmp/claudex-prompts && pwd -W 2>/dev/null || pwd)"
ls -t "$prompt_root" | head -1
# Open both files in that <call_id> dir:
#   - research prompt: mission context, orientation, seed path:line hits, output contract, no implementation rules
#   - implementation prompt: contains the research findings (event names / mechanism)

# Nothing escaped the sandbox.
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

In the transcript, confirm: no Agent tool call; the research Bash call carries `--sandbox read-only`; the implementation call launched only after the research call returned.

## Failure modes to watch for

- Orchestrator spawns a Claude subagent (Explore/general-purpose) for the exploration → hard regression; this is the exact behaviour the research path exists to replace.
- Orchestrator reads all 11 sandbox files into its own context instead of delegating → the quick-lookup exception is over-firing; it is meant for "a grep or two / ≤2 known files", not for mapping a directory of modules.
- Research call uses `--sandbox workspace-write` → wrong sandbox; it also drags the trust/bypass machinery into a job that must not have it.
- Research call gets `-c model_reasoning_effort=high` without `--effort` → research auto-escalation is forbidden.
- Research call retried with `--dangerously-bypass-approvals-and-sandbox` → the bypass fallback is reserved for `workspace-write` subtasks; on read-only it must never fire.
- Research prompt reuses the implementation must-include block ("stop after applying changes", formatter/test rules) → wrong template.
- Implementation prompt contains no trace of the research findings → the research round-trip was wasted; sequencing or embedding is broken.
- Implementation launches before research returns → sequencing violation (findings-consuming subtasks must wait).

## Cleanup

```bash
rm -rf tests/sandboxes/12-research
# Per-call subdirectories under /tmp/claudex-prompts/ are intentionally not cleaned —
# the OS temp cleanup handles them and the files are useful for debugging botched runs.
```
