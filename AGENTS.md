# Repository verification

For changes to the existing scripts, run:

```sh
zsh -n bin/claude-local bin/lms-load
bash -n bin/codex-local bin/codex-model-catalog bin/codex-export-models bin/codex-export-prompts bin/lms-load-codex
jq empty codex/models/local-models.json
bin/codex-model-catalog | jq empty
PYTHONPYCACHEPREFIX=/private/tmp/llms-pycache python3 -m py_compile bin/claude-local-sessions bin/lms-session-stats ollama/ollama-session-stats
for script in bin/* ollama/ollama-session-stats; do test -x "$script" || exit 1; done
```

No automated Markdown checker is configured. Review documentation changes directly and verify that referenced local paths exist.
