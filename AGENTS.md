# Repository verification

For changes to the existing scripts, run:

```sh
zsh -n bin/claude-local bin/lms-load
PYTHONPYCACHEPREFIX=/private/tmp/llms-pycache python3 -m py_compile bin/claude-local-sessions bin/lms-session-stats claude/ollama/ollama-session-stats
for script in bin/* claude/ollama/ollama-session-stats; do test -x "$script" || exit 1; done
```

No automated Markdown checker is configured. Review documentation changes directly and verify that referenced local paths exist.
