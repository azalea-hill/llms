# llms

Local-model setup for Claude Code on an Apple Silicon Mac (M5 Pro, 64 GB): LM Studio as the backend, the models, the launcher, and what was measured. Ollama was the first backend and is kept as an appendix.

```
bin/                       daily commands; each is symlinked into ~/.local/bin
  claude-local               launches Claude Code against LM Studio
  lms-load                   loads the main model plus the haiku model into LM Studio with explicit context targets
  lms-session-stats          per-session prefill, cache, and timing stats from LM Studio's log
  claude-local-sessions      lists a project's Claude Code sessions with the model each one used
ollama/                    the Ollama Modelfiles and stats script, kept for reference
```

Symlinks: `ln -sf ~/apps/ah/llms/bin/<name> ~/.local/bin/<name>` for each file in `bin/`.

## Workflow

1. Normal `claude` session on fable or opus. Write the plan for a subtask to a file, for example `plans/<task>.md`. Make it self-contained: files to touch, the exact change, how to verify. The local session starts with zero shared context.
2. `claude-local` session in the same repo. First message: `Implement plans/<task>.md. Read it before doing anything else.`
3. Review the diff yourself or in the Anthropic session.

Sessions never mix cloud and local. Claude Code sends every request in a session to one `ANTHROPIC_BASE_URL`, and the `ANTHROPIC_DEFAULT_*_MODEL` variables only change the model name in the request.

## Models

| Role            | LM Studio key                      | Size  | Notes                                                                         |
| --------------- | ---------------------------------- | ----- | ----------------------------------------------------------------------------- |
| main            | `qwen/qwen3.6-35b-a3b` (MLX 4-bit) | 20 GB | 35B mixture-of-experts, 3B active per token; the daily driver                 |
| main, alternate | `qwen/qwen3.8-27b` (MLX 4-bit)     | 16 GB | dense 27B; higher benchmark scores, several times slower                      |
| haiku           | `ibm/granite-4-micro` (GGUF)       | 2 GB  | background calls (session titles, summaries) and subagents; no reasoning mode |

```zsh
lms get --mlx qwen/qwen3.6-35b-a3b
lms get --mlx qwen/qwen3.8-27b
lms get ibm/granite-4-micro
lms ls
```

Model keys are the catalog names. `lms get <key>@<quant>` downloads another quantization under the same key (for example `@8bit`), and `lms ls --variants` lists every variant with the app's selected one starred. The CLI cannot choose a variant (lmstudio-bug-tracker #1462, open): `lms load <key>` always loads the variant selected in the app's model entry, so switching quantizations is done in LM Studio's UI, then `lms-load`. Sessions on either variant record the same model name; note which one was selected when comparing.

Why a mixture-of-experts model: token generation on this Mac is bound by memory bandwidth (307 GB/s), and every generated token streams the active weights through the GPU once. A dense 27B at 4-bit reads ~15 GB per token, ceiling about 20 tok/s; a 3B-active MoE reads ~2 GB, so 70-90 tok/s. The 35B total still has to be resident, which is what the 64 GB is for.

Why Granite for haiku: Claude Code's session-title call hardcodes `effort: high`. LM Studio maps request-level effort into the model's reasoning setting, and the request wins over the model's default, so a haiku model with a thinking mode (Gemma 4 12B was tried) reasons at length about a title. Granite has no reasoning mode, so there is nothing to turn on. It is also what LM Studio's own Claude Code guide uses. Ollama's `ollama launch claude` maps haiku to the main model instead; that works too and is the fallback if Granite trips on the title call's JSON schema.

## LM Studio setup

Install LM Studio and its CLI (`lms` is bundled; the app adds `~/.lmstudio/bin` to the path, or run `~/.lmstudio/bin/lms bootstrap`). Downloads live under `~/.lmstudio/models/`. LM Studio runs MLX models through mlx-lm and GGUF through llama.cpp, renders prompts with each model's own chat template, and exposes an Anthropic-compatible `/v1/messages` on port 1234.

```zsh
lms server start
lms-load
claude-local
```

`lms-load [model-key]` unloads everything, then loads the main model (the argument, else `LMS_MAIN`, else the 3.6 MoE) with a 128K context target, plus the haiku model (`LMS_SMALL`, default Granite) at 32K. It also passes `--speculative-draft-mtp`, which asks for the Qwen models' multi-token-prediction draft head; LM Studio does not log whether it engaged, and the 27B's decode rate on LM Studio (15-16 tok/s, below Ollama's 24-40 with MTP active) suggests it did not. Only one main model is resident at a time; LM Studio does not evict explicitly loaded models to make room, so switching is another `lms-load`. `lms ps` shows what is loaded with the effective context and the parallel slot count (`--parallel`, default 4: how many predictions the model runs concurrently; idle slots cost nothing); `lms unload --all` clears it. `lms load --estimate-only <key>` prints the memory a load would take.

Why explicit loads rather than on demand: LM Studio loads a model on demand when a request names one that is not loaded, but two settings in `~/.lmstudio/settings.json` make that wrong for this layout. `developer/unloadPreviousJITModelOnLoad` (on by default) evicts the previously on-demand-loaded model when another loads, so a background haiku call would unload the main model mid-session. `defaultContextLength` (custom, 8192 on this machine) is the context an on-demand load gets, far below the 131,072 the launcher declares, so the transcript would be silently truncated. On-demand loads also unload after an hour idle (`jitModelTTL`). Explicit loads bypass all three. Turning off the JIT-unload setting and raising the default context in the app (Developer tab) is a worthwhile safety net.

Context length on the MLX engine is a target, not a cap: LM Studio computes the largest context whose KV fits under its memory ceiling (`context_fit` lines in the server log; 208,384 for the 27B, 262,144 for smaller models on this machine) and uses that as the effective context, allocating KV lazily. All effective values sit above the 131,072 the launcher declares, so Claude Code compacts first and nothing is truncated.

Sampling defaults are per model and live in the app: My Models, the model's gear, Inference tab. The catalog entry for `qwen/qwen3.6-35b-a3b` already ships Qwen's coding preset (temperature 0.6, top-k 20, top-p 0.95, min-p 0); the model's own `generation_config.json` says temperature 1.0, so the 0.6 is catalog metadata. Repeat penalty defaults to disabled, which equals 1.0. Presence penalty is not exposed for the MLX engine and is effectively 0. Nothing needs changing; LM Studio would write any deviation from the catalog defaults to `~/.lmstudio/.internal/user-concrete-model-default-config/<key>.json`. Claude Code sends no sampling parameters, so these defaults are what run. Per-model reasoning lives in the same panel: for Gemma 4 it is "Custom fields, Enable thinking"; "Reasoning Section Parsing" only controls how `<think>` output is split for display.

## The `claude-local` launcher

`bin/claude-local` checks that LM Studio is answering, exports the variables below, prints the session ID and model mapping, and execs `claude` with any remaining arguments.

```zsh
claude-local
LMS_MAIN=qwen/qwen3.8-27b claude-local
claude-local --resume <id>
```

Variables:

```zsh
ANTHROPIC_BASE_URL=http://localhost:1234
ANTHROPIC_AUTH_TOKEN=lmstudio
ANTHROPIC_API_KEY=
ANTHROPIC_MODEL=opus
ANTHROPIC_DEFAULT_FABLE_MODEL=$LMS_MAIN
ANTHROPIC_DEFAULT_OPUS_MODEL=$LMS_MAIN
ANTHROPIC_DEFAULT_SONNET_MODEL=$LMS_MAIN
ANTHROPIC_DEFAULT_HAIKU_MODEL=$LMS_SMALL
CLAUDE_CODE_SUBAGENT_MODEL=haiku
CLAUDE_CODE_TOTAL_TOKENS_REMINDER=off
CLAUDE_CODE_ATTRIBUTION_HEADER=0
CLAUDE_CODE_MAX_CONTEXT_TOKENS=131072
CLAUDE_CODE_DISABLE_1M_CONTEXT=1
CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70
```

- `ANTHROPIC_BASE_URL` sends all inference to LM Studio (override with `LMS_URL`). Nothing in the session reaches Anthropic.
- `ANTHROPIC_AUTH_TOKEN` with an empty `ANTHROPIC_API_KEY` satisfies Claude Code's credential check; LM Studio ignores the value unless Require Authentication is on. A token replaces the claude.ai login for the session, which is what you want here. Normal `claude` sessions are unaffected because the variables are set only inside the launcher process.
- The three main aliases all resolve to `LMS_MAIN` so `/model` cannot land on an unloaded model; `LMS_MAIN` is the one thing to change to compare models. Haiku is `LMS_SMALL`. Claude Code uses haiku for background work (titles, summaries); `CLAUDE_CODE_SUBAGENT_MODEL=haiku` would send subagents there too, but the launcher's tool set omits `Agent`, so no subagents run unless `--tools` is overridden.
- `CLAUDE_CODE_TOTAL_TOKENS_REMINDER=off` stops Claude Code from appending a `role: "system"` message every turn (`<total_tokens>N tokens left</total_tokens>`, a budget note). Both LM Studio's Anthropic adapter and Ollama's qwen3.8 renderer fold mid-conversation system messages into the leading system turn, so each new one shifted every later token and destroyed the prefix cache on every request. Ollama's `ollama launch claude` sets this variable for the same reason (its v0.33.0 release notes); it is undocumented on the Claude Code side and was verified here against a fake upstream. `CLAUDE_CODE_ATTRIBUTION_HEADER=0` comes from the same place.
- `CLAUDE_CODE_MAX_CONTEXT_TOKENS=131072` tells Claude Code the window for an unrecognized model ID so it compacts in time; `CLAUDE_CODE_DISABLE_1M_CONTEXT=1` makes that apply even when the account's default model alias carries a `[1m]` suffix. `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` compacts at 70% of the window; compaction is itself a model request over the whole transcript, so on a local model it should happen with headroom.

Flags the launcher adds unless you pass your own: `--effort medium` (a flag rather than `CLAUDE_CODE_EFFORT_LEVEL`, because the env var is a hard override that `/effort` cannot change mid-session), `--permission-mode acceptEdits` (auto mode cannot work locally: its classifier requests follow `ANTHROPIC_BASE_URL` and need a Claude model), `--tools Bash,Read,Edit,Write,Glob,Grep,AskUserQuestion`, `--strict-mcp-config`, and a `--session-id` (skipped when resuming). The tool restriction matters for context: on a non-first-party base URL Claude Code sends every tool definition in full on every request, and the default built-in set is 21 tools and about 46 KB of JSON before `Artifact` and any connector tools; the six-tool set is about 10 KB. A local model executing a plan needs none of the rest.

Effort: Claude Code sends `output_config: {"effort": ...}` and LM Studio maps it into the model's reasoning setting. Qwen3.8 has low/medium/xhigh levels and honors it. Qwen3.6 only has on/off, so any level becomes `on` (the log warns and falls back; harmless). `/effort` changes it mid-session.

`/context` in a local session is an estimate, not a count: LM Studio has no `count_tokens`, so Claude Code falls back to a character-based guess that runs about 1.3x above what Qwen's tokenizer produces. Compaction triggers off the estimate, so it fires early rather than late.

## Seeing what LM Studio is doing

`lms ps` for what is loaded. `lms log stream` for live requests and responses; the files are under `~/.lmstudio/server-logs/`. Verbose logging (on by default) writes every request body to the log, which is useful for diagnosis and large.

`lms-session-stats [HH:MM] [session-id]` parses the log into a per-turn table: context, cached and new prompt tokens, prefill time, total time, and with a Claude Code session ID, output tokens per turn and decode rate from the transcript (per-turn alignment can be off by one; the totals are reliable). The `cached` column should climb with the conversation; if it sits flat while `new` stays large, the prefix cache is not holding. `claude-local-sessions [dir]` lists a project's sessions newest first with the models each used and the first prompt; `claude-local --resume <id>` continues one.

Two log lines that are expected: `Unexpected endpoint or method. (HEAD /api/hello). Returning 200 anyway` is Claude Code's connection warm-up probe, logged at ERROR level but answered correctly. `Reasoning setting 'medium' is not supported by model 'qwen/qwen3.6-35b-a3b' ... falling back to 'on'` is the effort mapping on a model without levels.

One known LM Studio bug: `POST /v1/messages/count_tokens` returns HTTP 200 with an error body (lmstudio-bug-tracker #2055). Claude Code appears to retry that with growing backoff ("Waiting for API response, will retry"); it was seen once, in the session where every turn was also re-prefilling, and not since. Ollama returns 404, which Claude Code handles by falling back to an estimate. If the banner comes back, a pass-through that answers that one path with 404 is the workaround, pending the LM Studio fix.

## Memory

macOS lets Metal address about 75% of unified memory, 51.8 GB on this Mac, and model weights loaded for Metal are wired and never page out; whatever they take comes out of what Docker, Chrome, VS Code, and the database client have left. With the normal engineering stack running, free memory is about 20-25 GB before any model, and the LM Studio server log's `context_fit` line records the working set it saw at load. The 4-bit MoE plus Granite is about 22 GB of weights plus KV and fits alongside the stack on a normal day; the 8-bit MoE (~37 GB) needs the machine to itself. Docker Desktop's memory limit is a VM allocation set in its settings and is the biggest single lever. Do not raise `iogpu.wired_limit_mb` while the engineering stack is running; it takes memory from the apps.

## Measurements

PR-summarization prompt against the mymassgov repo (about 9K real first-turn tokens with the trimmed tool set), 2026-09-11/12. Time is first user message to final answer.

| | Opus 5 | qwen3.6 4-bit, LM Studio | qwen3.6 8-bit, LM Studio | qwen3.6 4-bit, Ollama | qwen3.8 4-bit, LM Studio | qwen3.8 4-bit, Ollama |
| --- | --- | --- | --- | --- | --- | --- |
| Time to answer | 45 s | 59 s | 46 s | 104 s | 458 s | 699 s |
| Requests / tool calls | 4 / 3 | 7 / 12 | 3 / 6 | 10 / 18 | 9 / 11 | 8 / 14 |
| First-turn prefill | n/a | 1,924 tok/s | 1,594 tok/s | 978 tok/s | 347 tok/s | 366 tok/s |
| Decode | n/a | ~71-74 tok/s | 44-53 tok/s | 48-85 tok/s | ~15-16 tok/s | 24-40 tok/s |
| Tokens prefilled | n/a | 24K | 19K | 26K | 26.5K | 175K (cache broken) |
| Output tokens | 4,382 | 3,194 | 1,802 | 4,476 | 5,410 | 6,906 |
| Final answer | 5,531 chars | 6,260 | 4,434 | 7,416 | 5,806 | 6,627 |

One run per configuration, and the models vary their tool-call plans between runs, so wall-clock and tool-call counts are single samples; prefill and decode rates are the stable signals.

Quality on this task: on Ollama, the 3.6 and 3.8 Qwens at 4-bit and 8-bit were judged indistinguishable and close to Opus, and Gemma 4 26B-A4B significantly weaker. On LM Studio, the 8-bit 3.6's answer was judged slightly better than the 4-bit's, but it investigated less (6 tool calls against 12, 19K of context read against 25K); the 4-bit stays the default for fitting alongside the engineering stack and decoding ~30% faster, the 8-bit is the choice when the machine is free. The dense 27B has the better benchmark numbers and may pull ahead on complex agentic work; untested. Simple-prompt decode with nothing else running, on Ollama: 3.6 MoE 4-bit ~89 tok/s and 8-bit ~81; 3.8 27B 4-bit 34 tok/s and 8-bit ~15-20 (the dense model is bandwidth-bound, so 8-bit halves it).

The Ollama qwen3.8 row is from before `CLAUDE_CODE_TOTAL_TOKENS_REMINDER=off` existed in the launcher; the 175K of re-prefill was the folding problem, not the runtime.

## On a 48 GB Mac

Metal gets about 36 GB there by default, 40 GB with `sudo sysctl iogpu.wired_limit_mb=40960`. The 4-bit MoE plus Granite fits with room; the 8-bit MoE does not fit next to a haiku model. A 48 GB M4 Max has more bandwidth than the M5 Pro (546 vs 307 GB/s): the 4-bit MoE measured ~116 tok/s decode there against ~89 on the M5 Pro. Without the M5 neural accelerators the first-turn prefill will be two to three times slower.

## Codex as the client

Codex CLI speaks the OpenAI API, which LM Studio and `mlx_lm.server` both expose, so it can sit directly in front of either with no Anthropic translation. `~/.codex/config.toml` takes a custom provider with a `base_url` and, for servers that only implement chat completions, `wire_api = "chat"`. LM Studio serves both clients from one loaded model, so the cheap comparison is Claude Code and Codex against the same LM Studio instance on the same plan file.

## Ollama (kept for reference)

Ollama was the first backend. `ollama/modelfiles/` holds one Modelfile per model name the old launcher used; each pins `num_ctx` and, for the Qwen3.6 builds, Qwen's coding sampling (`temperature 0.6`, `presence_penalty 0`, which Ollama's plain tags do not set). `ollama create <name> -f ollama/modelfiles/<name>` bakes one into Ollama's store. `ollama/ollama-session-stats [logfile] [HH:MM]` parses `~/.ollama/logs/server.log` into the same per-turn table as `lms-session-stats`.

To use Ollama again: `ANTHROPIC_BASE_URL=http://localhost:11434`, `ANTHROPIC_AUTH_TOKEN=ollama`, the model names from the Modelfiles, and the same `CLAUDE_CODE_*` variables as the launcher. Ollama's own `ollama launch claude` sets the same environment.

What was learned on Ollama, still true:

- Ollama's MLX runner is used by the `-mlx` and `-nvfp4` tags; the bare size tags are GGUF through llama.cpp and noticeably slower. `qwen3.6:35b-mlx` and `qwen3.6:35b-a3b-nvfp4` are the same blob; likewise `qwen3.8:27b-mlx` and `qwen3.8:27b-nvfp4`.
- Ollama's `qwen3.8` renderer (`model/renderers/qwen35.go`, `normalizeQwen38Messages`) folds mid-conversation system messages into the leading system turn, which broke the prefix cache every turn with Claude Code until `CLAUDE_CODE_TOTAL_TOKENS_REMINDER=off`. The same file inserts no reasoning instruction for `medium` on qwen3.8, so only `low` reduces its thinking there.
- Ollama evicts loaded models automatically to fit memory and logs `system memory ... free=` at every load; memory pressure from the engineering stack cut Qwen prefill 3-6x. `OLLAMA_KEEP_ALIVE` defaults to 5 minutes; `launchctl setenv OLLAMA_KEEP_ALIVE 2h` and relaunching the app keeps a model resident across a break.
- Ollama returns 404 for `count_tokens`, so Claude Code's estimate fallback works without help.

## Reference

- Claude Code model config and alias variables: https://code.claude.com/docs/en/model-config
- Claude Code env vars: https://code.claude.com/docs/en/env-vars
- Claude Code gateway compatibility guide: https://code.claude.com/docs/en/llm-gateway-protocol
- LM Studio Claude Code integration: https://lmstudio.ai/blog/claudecode
- LM Studio CLI: https://lmstudio.ai/docs/cli
- LM Studio count_tokens bug: https://github.com/lmstudio-ai/lmstudio-bug-tracker/issues/2055
- LM Studio variant loading bug: https://github.com/lmstudio-ai/lmstudio-bug-tracker/issues/1462
- Ollama launch integration (source of the two CLAUDE_CODE variables): https://github.com/ollama/ollama/blob/main/cmd/launch/claude.go
- Ollama qwen3.5/3.8 renderer: https://github.com/ollama/ollama/blob/main/model/renderers/qwen35.go
- Qwen3.6-35B-A3B model card (sampling settings): https://huggingface.co/Qwen/Qwen3.6-35B-A3B
- Qwen3.8-27B model card (sampling settings, reasoning effort): https://huggingface.co/Qwen/Qwen3.8-27B
