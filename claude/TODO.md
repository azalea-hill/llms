# TODO

## Web search

Claude Code's `WebSearch` tool executes via Anthropic's API and cannot be pointed elsewhere. Need a local alternative:

- Build a local MCP server that handles web search (e.g., via a local search API or scraping backend)
- Wire it into `claude-local`'s tool list so it appears as a local tool
- Verify it works with LM Studio's Anthropic adapter

Priority: medium. Useful but not blocking — the current `WebFetch` workaround (fetch + haiku summary) covers most cases.

## Script error handling

The scripts mostly rely on errors from the commands they invoke and do not consistently add actionable context:

- `lms-load` exits when `lms unload` or `lms load` fails, but does not add troubleshooting guidance
- `claude-local` doesn't verify Claude Code is installed before exec'ing it
- `lms-session-stats` and `claude-local-sessions` have no error handling on missing files
- No usage messages or argument validation

## Centralize Claude model config

`qwen/qwen3.6-35b-a3b` and `google/gemma-4-26b-a4b-qat` are hardcoded across `claude-local` and `lms-load`. If a better model comes out, you have to touch multiple files. A single source of truth (env var convention, config file, or shared script) would help.
