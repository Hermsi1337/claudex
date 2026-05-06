# 04 — `--model` flag override

## Goal

Verify that `/claudex --model <name> <task>` parses the flag, strips it from the task description, and forwards `--model <name>` to the actual `codex exec` call.

## Setup

```bash
rm -rf tests/sandboxes/04-model
mkdir -p tests/sandboxes/04-model
```

## Invocation

Substitute `<your-codex-model>` with a model name your local Codex CLI accepts (e.g. one shown by `codex --help` or whatever your `~/.codex/config.toml` references):

```
/claudex --model <your-codex-model> In tests/sandboxes/04-model/, create hello.py with a function say_hi() that returns the string "hi". Do not modify anything outside tests/sandboxes/04-model/.
```

## Expected behaviour

- **Argument parsing:** Claude removes `--model <your-codex-model>` from the natural-language task before composing the Codex prompt. The Codex prompt should not contain the literal string `--model`.
- **Phase 3:** Claude first writes the prompt to `/tmp/claudex-prompts/<slug>-<id>.md` (Write tool) and then issues a Bash call of the shape:
  ```bash
  codex exec --sandbox workspace-write --model <your-codex-model> --cd tests/sandboxes/04-model - < /tmp/claudex-prompts/<slug>-<id>.md
  ```
  Inspect the actual Bash command in the transcript — the `--model <your-codex-model>` segment must be there, and the prompt must come from the stdin redirect (no inline-quoted prompt, no heredoc). Order of flags may vary; what matters is that `--sandbox workspace-write`, `--model <your-codex-model>`, `--cd …`, and `- < <prompt-file>` are all present.
- **Result:** `hello.py` is created with a `say_hi()` returning `"hi"`. Whether the model behaved better/worse is not what we're testing — only that the override travelled through.

## Verify

```bash
cat tests/sandboxes/04-model/hello.py
python -c "
import sys; sys.path.insert(0, 'tests/sandboxes/04-model')
from hello import say_hi
assert say_hi() == 'hi'
print('say_hi(): OK')
"

# nothing outside the sandbox
git status --porcelain -- ':(exclude)tests/sandboxes/'
# expected: empty
```

If you couldn't see the `codex exec ... --model <your-codex-model>` Bash call in the transcript (e.g. Claude Desktop hides the command), re-run with `--model definitely-not-a-real-model-xyz` and expect Codex to error out with a model-not-found message — that's a positive signal that the flag arrived.

## Cleanup

```bash
rm -rf tests/sandboxes/04-model
```
