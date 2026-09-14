# TODO

## Model research

Benchmark viable model sets for:

- **M5 Max, 128 GB**
- **M5 Ultra, 256 GB**
- **M5 Ultra, 512 GB**

Current pair (Qwen 35B + Gemma 26B, ~35 GB loaded) is tuned for 48 GB. What model combinations make sense at higher memory ceilings? Consider dense, MoE, and MLX-quantized variants. Target: 200K context per model, cache-friendly loading strategy.

## Web search

Claude Code's `WebSearch` tool executes via Anthropic's API and cannot be pointed elsewhere. Need a local alternative:

- Build a local MCP server that handles web search (e.g., via a local search API or scraping backend)
- Wire it into `claude-local`'s tool list so it appears as a local tool
- Verify it works with LM Studio's Anthropic adapter

Priority: medium. Useful but not blocking — the current `WebFetch` workaround (fetch + haiku summary) covers most cases.

## Script error handling

All scripts fail silently or with cryptic tracebacks when things go wrong:

- `lms-load` doesn't check if `lms unload` or `lms load` succeeded
- `claude-local` doesn't verify Claude Code is installed before exec'ing it
- `lms-session-stats` and `claude-local-sessions` have no error handling on missing files
- No usage messages or argument validation

## Centralize model config

`qwen/qwen3.6-35b-a3b` and `google/gemma-4-26b-a4b-qat` are hardcoded across `claude-local` and `lms-load`. If a better model comes out, you have to touch multiple files. A single source of truth (env var convention, config file, or shared script) would help.

## LM Studio settings helper

The README recommends changing `~/.lmstudio/settings.json` (turning off `developer/unloadPreviousJITModelOnLoad`, raising `defaultContextLength`) but there's no helper to do that. A one-liner script would save manual editing.

## Model version pinning

Model names are strings, not pinned versions. A better model could overwrite the same HuggingFace key and break things. Pin to specific revision hashes or use `lms get <key>@<revision>` to lock versions.

## Clean up or document `ollama/`

Modelfiles and `ollama-session-stats` exist but aren't referenced in README or TODO. They're artifacts from the Ollama-first approach and are now dead code unless you want to keep them as reference. Either remove them or document their purpose.

## No "undo" command

You can `lms unload --all` but there's no "go back to defaults" or "unload all and restore to idle" command. Useful after experimenting with different models.
