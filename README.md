# llms

Local-model coding-agent setups for Apple Silicon, with LM Studio as the
inference backend.

## Setups

- [Claude Code](claude/README.md) is the existing three-model setup. Its model
  notes and archived Ollama configuration live under `claude/`.
- [Codex](CODEX-HYBRID-PLAN.md) is currently an implementation plan for a
  primarily local setup with an isolated OpenAI planning profile and a narrow
  web-search MCP bridge.

## Existing commands

The top-level `bin/` directory contains the user-facing scripts for both setups.
Existing symlinks or PATH entries that point there continue to work:

```zsh
lms-load
claude-local
lms-session-stats
claude-local-sessions
```

## Repository plans and backlog

- [Hybrid local Codex implementation plan](CODEX-HYBRID-PLAN.md)
- [Cross-cutting backlog](TODO.md)
- [Claude-specific backlog](claude/TODO.md)
