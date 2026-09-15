# Archived Ollama experiments

This directory preserves the repository's earlier Ollama-based model definitions and log parser for reference. It is not part of the current Claude Code or Codex setup; both current launchers use LM Studio.

- `modelfiles/` contains the old Ollama model aliases and context settings.
- `ollama-session-stats [log-file] [HH:MM]` parses an Ollama server log from that implementation. With no path, it reads `~/.ollama/logs/server.log`.

Do not create these aliases or start Ollama when following the current setup guides.
