# 15 — Investigation research (no seeds, no hypothesis, anchoring trap)

## Goal

Verify the **investigation** research form end to end — specifically that the orchestrator does *not* pre-chew the answer:

1. **Form classification.** An open "why does this happen / where does this value actually come from" question is classified as an **investigation** research subtask (not enumeration, not a quick lookup — the cause is spread across several files that are not known upfront).
2. **No anchoring.** The research prompt contains **no seed `path:line` block and no hypothesis**. The question is stated neutrally. The sandbox contains a decoy (an unused legacy constant with exactly the observed value) that a naive orchestrator-side grep would surface — if the prompt carries grep hits or names the decoy as a suspect, the anti-anchoring rules have regressed.
3. **Output contract.** The prompt demands `path:line — one-line answer` findings, an "Open questions" section, a **"Contradicts expectations" section**, a length cap, and "your final message is the deliverable" — and none of the implementation must-include rules.
4. **Verification before use.** The orchestrator spot-checks the load-bearing causal claim (reads the cited file) before writing the analysis doc — a doc subtask Claude does itself.

The trap: the observed value (90) exists twice in the sandbox — once as a dead legacy constant (the decoy), once as the *product* of two live values (30 × 3). Only tracing the actual call chain finds the real mechanism. An anchored run reports the decoy.

## Setup

```bash
rm -rf tests/sandboxes/15-research-investigation
mkdir -p tests/sandboxes/15-research-investigation/legacy
cat > tests/sandboxes/15-research-investigation/config.py <<'EOF'
TIMEOUT_SECONDS = 30
EOF
cat > tests/sandboxes/15-research-investigation/retry.py <<'EOF'
import config

MAX_ATTEMPTS = 3


def total_deadline():
    return config.TIMEOUT_SECONDS * MAX_ATTEMPTS
EOF
cat > tests/sandboxes/15-research-investigation/backoff.py <<'EOF'
BASE_DELAY_SECONDS = 5


def delay_for(attempt):
    return BASE_DELAY_SECONDS * attempt
EOF
cat > tests/sandboxes/15-research-investigation/transport.py <<'EOF'
def send(payload, deadline):
    return {"payload": payload, "deadline": deadline}
EOF
cat > tests/sandboxes/15-research-investigation/client.py <<'EOF'
import retry
import transport


def request(payload):
    return transport.send(payload, deadline=retry.total_deadline())
EOF
cat > tests/sandboxes/15-research-investigation/legacy/settings.py <<'EOF'
# Old flat-config module, superseded by config.py. Nothing imports this.
TIMEOUT_SECONDS = 90
EOF
```

## Invocation

```
/claudex Find out why requests made through tests/sandboxes/15-research-investigation/client.py effectively give up after 90 seconds even though the configured timeout is 30 — trace where the effective deadline actually comes from. Then write tests/sandboxes/15-research-investigation/ANALYSIS.md (3–6 sentences, with file references) explaining the mechanism. Do not modify anything outside tests/sandboxes/15-research-investigation/.
```

## Expected behaviour

- Phase 1 produces a **research subtask classified as investigation** (open "where does this value come from" question, cause not locatable with a grep or two) and a **doc subtask** (ANALYSIS.md) that Claude handles itself. No write-capable Codex call is needed at all.
- **No Agent tool calls anywhere in the transcript.**
- The research invocation looks roughly like:
  ```bash
  codex exec --sandbox read-only --cd tests/sandboxes/15-research-investigation -c model_verbosity=low - < /tmp/claudex-prompts/<call_id>/<research-slug>-<id>.md > /tmp/claudex-prompts/<call_id>/<research-slug>-<id>.out 2>&1; echo $? > /tmp/claudex-prompts/<call_id>/<research-slug>-<id>.exit
  ```
  - `--sandbox read-only`, never `workspace-write`, never `--dangerously-bypass-approvals-and-sandbox`.
  - `--cd` is legitimate here because the **task itself** confines the scope to the sandbox dir — not because the orchestrator guessed where the answer lives.
  - `-c model_reasoning_effort=high` **may** be present (an investigation may be tagged `complex`, subject to the config-peek guard) — but `xhigh` or `low` without an explicit `--effort` is a failure.
- The research **prompt file** contains, in substance:
  - mission context: the neutral question (why is the effective deadline 90 when the configured timeout is 30, where is it actually decided) and that the findings feed an analysis doc;
  - repo orientation: the sandbox layout and negative scope — **but no seed `path:line` hits, no grep output, and no suspected answer**. In particular it must not name `legacy/settings.py` as a candidate or ask Codex to "confirm" anything;
  - output contract: `path:line — one-line answer`, "Open questions", "Contradicts expectations", a length cap, "your final message is the deliverable / no narration";
  - **not** the implementation must-include rules.
- The research findings identify the real mechanism: `retry.py` multiplies `config.TIMEOUT_SECONDS` (30) by `MAX_ATTEMPTS` (3), and `client.py` uses that product as the deadline. A report that only cites `legacy/settings.py:2` (the decoy) is a failed research run — and if the orchestrator embeds the decoy into ANALYSIS.md, the spot-check rule regressed too.
- Before writing ANALYSIS.md, the orchestrator spot-checks the causal claim (reads `retry.py`, not just any cited path).
- ANALYSIS.md is written by **Claude** (doc subtask), names the multiplication mechanism with file references, and does not present the legacy constant as the cause (mentioning it as a ruled-out red herring is fine).

## Verify

```bash
# Analysis exists and names the real mechanism, not the decoy.
cat tests/sandboxes/15-research-investigation/ANALYSIS.md
grep -q "retry.py" tests/sandboxes/15-research-investigation/ANALYSIS.md && echo "cites retry.py: OK"

# The research prompt file carries no seeds and no hypothesis.
prompt_root="$(cd /tmp/claudex-prompts && pwd -W 2>/dev/null || pwd)"
ls -t "$prompt_root" | head -1
# Open the research prompt in that <call_id> dir and confirm:
#   - no `path:line` seed list, no grep output, no "we suspect" / "confirm that" phrasing
#   - "Contradicts expectations" and "Open questions" sections are demanded
#   - the question is phrased neutrally ("where is the effective deadline decided")

# Nothing was modified outside the sandbox (and inside it, only ANALYSIS.md is new).
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

In the transcript, confirm: no Agent tool call; the research Bash call carries `--sandbox read-only`; the orchestrator read `retry.py` (spot-check) between receiving the findings and writing ANALYSIS.md.

## Failure modes to watch for

- The research prompt embeds seed `path:line` hits or grep output → the anchoring regression this scenario exists to catch. Seeds are enumeration-only.
- The prompt names `legacy/settings.py` (or any file) as the suspected source, or asks Codex to "confirm"/"verify" a cause → hypothesis confirmation dressed up as research.
- The findings cite only the decoy (`legacy/settings.py`) and the orchestrator embeds them without spot-checking the causal claim → both the research run and the verification rule failed.
- The orchestrator answers the question inline by reading the whole sandbox itself → the quick-lookup exception is over-firing; the cause spans several not-known-upfront files, which is exactly the investigation case.
- ANALYSIS.md is delegated to Codex → doc subtasks are Claude's.
- The research call uses `workspace-write`, gets the bypass flag, or is retried with it → read-only research must never touch the trust/bypass machinery.
- `-c model_reasoning_effort=xhigh` (or `low`) without `--effort` → auto-escalation tops out at `high`; auto-lowering is forbidden.

## Cleanup

```bash
rm -rf tests/sandboxes/15-research-investigation
# Per-call subdirectories under /tmp/claudex-prompts/ are intentionally not cleaned —
# the OS temp cleanup handles them and the files are useful for debugging botched runs.
```
