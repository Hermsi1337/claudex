# 08 — Prompt-via-stdin (shell-escaping safety)

## Goal

Verify that the orchestrator delivers Codex prompts via a file + stdin redirect (`codex exec [flags] - < /tmp/claudex-prompts/<call_id>/<slug>-<id>.md`, plus the `.out`/`.exit` sentinel redirects), **not** as an inline-quoted positional argument and **not** through a heredoc. The prompt content for this scenario is deliberately full of shell metacharacters that would have broken inline quoting.

## Setup

```bash
rm -rf tests/sandboxes/08-stdin
mkdir -p tests/sandboxes/08-stdin
```

## Invocation

The task description below contains backticks, `$VAR`-looking tokens, embedded double **and** single quotes, and the literal token `EOF` — anything that is supposed to land in the prompt body and travel safely to Codex.

```
/claudex In tests/sandboxes/08-stdin/, create a file notes.md whose body is exactly the following three lines (no extra leading/trailing whitespace, no extra lines): the first line `EOF` between backticks, the second line $HOME without expansion, the third line a sentence that uses both 'single' and "double" quotes. Then create scan.py: a function find_eof(path) that reads the file at path and returns True iff any line equals the literal three characters E, O, F (no backticks). Do not modify anything outside tests/sandboxes/08-stdin/.
```

## Expected behaviour

- **Phase 3 — prompt delivery:**
  - Before invoking Codex, Claude resolves a per-call subdirectory under `/tmp/claudex-prompts/` (`<call_id>` = timestamp + 6 hex chars) and calls the **Write** tool to create a file inside it, e.g. `/tmp/claudex-prompts/<call_id>/notes-and-scan-<id>.md`. The Write tool is what materialises the prompt — **not** a `cat <<EOF` heredoc inside Bash.
  - The actual `codex exec` Bash call has the form:
    ```bash
    codex exec --sandbox workspace-write --cd tests/sandboxes/08-stdin - < /tmp/claudex-prompts/<call_id>/<slug>-<id>.md > /tmp/claudex-prompts/<call_id>/<slug>-<id>.out 2>&1; echo $? > /tmp/claudex-prompts/<call_id>/<slug>-<id>.exit
    ```
    `-` (read prompt from stdin) and `< <file>` (feed stdin from the prompt file) are both required, as are the `.out`/`.exit` sentinel redirects (scenario 16 covers those in depth). No part of the natural-language prompt may appear as a positional argument or inside a heredoc.
- **Phase 4 — review:** confirms `notes.md` contains exactly the three lines specified, with backticks, `$HOME` (literal, unexpanded), and the mixed-quote sentence intact. Confirms `scan.py` defines `find_eof` correctly.

## Failure mode to watch for

If the orchestrator takes a shortcut and inlines the prompt as `codex exec ... "..."`, the call will likely either:

- die during shell parsing (unbalanced quotes), or
- silently corrupt the prompt (e.g. `$HOME` getting expanded by bash to your real home dir before Codex ever sees it), or
- misbehave because the literal `EOF` in the prompt collided with a heredoc delimiter.

Any of those is a **regression on the prompt-delivery rule** and must be flagged.

## Verify

```bash
# The prompt file must exist (the orchestrator should not delete it).
# Find the most recent per-call subdirectory under the resolved prompt root and check it has a .md inside.
prompt_root="$(cd /tmp/claudex-prompts && pwd -W 2>/dev/null || pwd)"
latest_call="$(ls -1t "$prompt_root" 2>/dev/null | head -1)"
if [ -n "$latest_call" ] && ls "$prompt_root/$latest_call"/*.md >/dev/null 2>&1; then
  echo "prompt file found in $prompt_root/$latest_call"
else
  echo "NO PROMPT FILE FOUND — regression"
fi

# notes.md content
cat tests/sandboxes/08-stdin/notes.md
# expected exact content (three lines):
#   `EOF`
#   $HOME
#   a sentence that uses both 'single' and "double" quotes.

# functional check on scan.py against a tiny throwaway input
python -c "
import sys; sys.path.insert(0, 'tests/sandboxes/08-stdin')
from scan import find_eof
import tempfile, os
with tempfile.NamedTemporaryFile('w', delete=False, suffix='.txt') as fh:
    fh.write('hello\nEOF\nworld\n'); p_yes = fh.name
with tempfile.NamedTemporaryFile('w', delete=False, suffix='.txt') as fh:
    fh.write('hello\nworld\n');       p_no  = fh.name
assert find_eof(p_yes) is True,  'find_eof failed positive case'
assert find_eof(p_no)  is False, 'find_eof failed negative case'
os.remove(p_yes); os.remove(p_no)
print('scan.find_eof: OK')
"

# nothing outside the sandbox
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

## Cleanup

```bash
rm -rf tests/sandboxes/08-stdin
# Optional — the per-call subdirectory under /tmp/claudex-prompts/ is left behind on purpose
# (debugging value). The OS temp cleanup handles it eventually.
```
