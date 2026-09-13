# llms

Local-model setup for Claude Code on an Apple Silicon Mac: LM Studio as the backend, two models, a launcher, and what was measured. Every request in a `claude-local` session stays on the machine.

[MODELS.md](MODELS.md) is the companion: which models were tried, what they cost in memory, and what to run on a 48, 64, 128, or 256 GB Mac. Ollama was the first backend and is kept as an appendix here.

```
bin/                       daily commands; put on PATH (see Getting started)
  claude-local               launches Claude Code against LM Studio
  lms-load                   loads the main model plus the haiku model with explicit context targets
  lms-session-stats          per-session prefill, cache, and timing stats from LM Studio's log
  claude-local-sessions      lists a project's Claude Code sessions with the model each one used
ollama/                    the Ollama Modelfiles and stats script, kept for reference
MODELS.md                  model selection, memory, and per-machine recommendations
```

## Getting started

Fifteen minutes, most of it the model download.

You need:

- **An Apple Silicon Mac with 48 GB of unified memory or more.** The two models want about 31 GB resident. 32 GB will not hold them next to a normal engineering stack — see [MODELS.md](MODELS.md#recommendations-by-machine).
- **About 23 GB of free disk** for the models.
- **[LM Studio](https://lmstudio.ai)**, the inference server.
- **[Claude Code](https://code.claude.com/docs/en/quickstart)**, the client. An Anthropic account is only needed for normal cloud sessions; `claude-local` authenticates against LM Studio and never prompts for a login.
- **zsh**, the macOS default shell. The scripts are `#!/bin/zsh`.

### 1. Install Claude Code

```zsh
curl -fsSL https://claude.ai/install.sh | bash    # or: brew install --cask claude-code
claude --version
```

### 2. Install LM Studio and its CLI

Download the app from [lmstudio.ai](https://lmstudio.ai) and launch it once. The `lms` CLI ships inside it, and the app adds `~/.lmstudio/bin` to your PATH; if `lms` is not found, bootstrap it yourself:

```zsh
~/.lmstudio/bin/lms bootstrap
lms version
```

### 3. Download the two models

```zsh
lms get --mlx qwen/qwen3.6-35b-a3b    # main, MLX 4-bit, 20.4 GB
lms get ibm/granite-4-micro           # haiku, GGUF, 2.1 GB
lms ls
```

`qwen/qwen3.6-35b-a3b` answers as fable, opus, and sonnet. `ibm/granite-4-micro` answers as haiku — the background calls Claude Code makes for session titles and summaries. [MODELS.md](MODELS.md) explains why this pair and what else was tried. Files land in `~/.lmstudio/models/`.

### 4. Put `bin/` on your PATH

Symlink each script into a directory already on your PATH:

```zsh
for f in ~/apps/ah/llms/bin/*; do ln -sf "$f" ~/.local/bin/"${f:t}"; done
```

Or add the directory itself, in `~/.zshrc`:

```zsh
export PATH="$HOME/apps/ah/llms/bin:$PATH"
```

Symlinks are what this repo uses. Either way the scripts run from the checkout, so edits take effect with no reinstall.

### 5. Start the server, load the models, run

```zsh
lms server start    # once per boot; listens on :1234
lms-load            # unloads everything, loads the pair; about 30 s
claude-local        # from whatever repo you want to work in
```

`claude-local` checks that LM Studio is answering, prints the session ID and the model mapping, and drops you at a normal Claude Code prompt:

```
claude-local [http://localhost:1234] session=8f3c... main=qwen/qwen3.6-35b-a3b haiku=ibm/granite-4-micro
```

Ask it something small first — the first turn of a session has to prefill the system prompt and tool definitions, so it is the slowest one. `lms ps` shows what is resident and at what context; `lms unload --all` frees it.

## Workflow

The local model is good at executing a written plan and weaker at deciding what the plan should be. So:

1. Normal `claude` session on fable or opus. Write the plan for a subtask to a file, for example `plans/<task>.md`. Make it self-contained: files to touch, the exact change, how to verify. The local session starts with zero shared context.
2. `claude-local` session in the same repo. First message: `Implement plans/<task>.md. Read it before doing anything else.`
3. Review the diff yourself or in the Anthropic session.

Sessions never mix cloud and local. Claude Code sends every request in a session to one `ANTHROPIC_BASE_URL`, and the `ANTHROPIC_DEFAULT_*_MODEL` variables only change the model name in the request.

## Loading models: `lms-load`

```zsh
lms-load                              # the defaults
lms-load qwen/qwen3.8-27b             # a different main model
LMS_SMALL=ibm/granite-4-micro lms-load
```

`lms-load [model-key]` unloads everything, then loads the main model — the argument, else `LMS_MAIN`, else `qwen/qwen3.6-35b-a3b` — at a 200K context target with `--gpu max`, plus the haiku model (`LMS_SMALL`, default `ibm/granite-4-micro`) at 32K. Only one main model is resident at a time; LM Studio does not evict explicitly loaded models to make room, so switching models is another `lms-load`.

It also passes `--speculative-draft-mtp`, which asks for the Qwen models' multi-token-prediction draft head. LM Studio does not log whether it engaged, and the dense 27B's decode rate here (15-16 tok/s, against 24-40 on Ollama with MTP active) suggests it did not.

**Why explicit loads rather than on demand.** LM Studio will load a model when a request names one that is not loaded, but two settings in `~/.lmstudio/settings.json` make that wrong for this layout. `developer/unloadPreviousJITModelOnLoad` (on by default) evicts the previously on-demand-loaded model when another loads, so a background haiku call would unload the main model mid-session. `defaultContextLength` is the context an on-demand load gets — 8192 on this machine, far below the 200,000 the launcher declares — so the transcript would be silently truncated. On-demand loads also unload after an hour idle (`jitModelTTL`). Explicit loads bypass all three. Turning off the JIT-unload setting and raising the default context in the app's Developer tab is a worthwhile safety net.

**Context length on MLX is a target, not a cap.** LM Studio computes the largest context whose KV cache fits under its memory ceiling (the `context_fit` lines in the server log: 208,384 for the dense 27B, 262,144 for smaller models on the 64 GB machine) and uses that as the effective context, allocating KV lazily as the conversation grows. All of those sit above the 200,000 the launcher declares, so Claude Code compacts before anything is truncated. GGUF through llama.cpp behaves differently — it preallocates, which is why Granite's estimated footprint moves with `--context-length` and the MoE's does not.

**Sampling defaults are per model and live in the app**, under My Models → the model's gear → Inference. The catalog entry for `qwen/qwen3.6-35b-a3b` already ships Qwen's coding preset (temperature 0.6, top-k 20, top-p 0.95, min-p 0); the model's own `generation_config.json` says temperature 1.0, so the 0.6 is catalog metadata. Repeat penalty defaults to disabled, which equals 1.0. Presence penalty is not exposed for the MLX engine and is effectively 0. Nothing needs changing. LM Studio writes any deviation from catalog defaults to `~/.lmstudio/.internal/user-concrete-model-default-config/<key>.json`. Claude Code sends no sampling parameters, so these defaults are what run. Per-model reasoning lives in the same panel; "Reasoning Section Parsing" only controls how `<think>` output is split for display.

**Switching quantization** is a UI operation. `lms get <key>@<quant>` downloads another quantization under the same key, `lms get --select <key>` opens variant selection, and `lms ls --variants` lists every variant with the app's selected one starred — but `lms load <key>` always loads the variant selected in the app's model entry ([lmstudio-bug-tracker #1462](https://github.com/lmstudio-ai/lmstudio-bug-tracker/issues/1462), open). So: pick the variant in LM Studio's UI, then `lms-load`. Sessions on either variant record the same model name, so note which one was selected when you compare.

## The launcher: `claude-local`

`bin/claude-local` checks that LM Studio is answering, exports the variables below, prints the session ID and model mapping, and execs `claude` with any remaining arguments.

```zsh
claude-local
LMS_MAIN=qwen/qwen3.8-27b claude-local
claude-local --resume <id>
```

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
CLAUDE_CODE_MAX_CONTEXT_TOKENS=200000
CLAUDE_CODE_DISABLE_1M_CONTEXT=1
CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70
```

- `ANTHROPIC_BASE_URL` sends all inference to LM Studio (override with `LMS_URL`). Nothing in the session reaches Anthropic.
- `ANTHROPIC_AUTH_TOKEN` with an empty `ANTHROPIC_API_KEY` satisfies Claude Code's credential check; LM Studio ignores the value unless Require Authentication is on. A token replaces the claude.ai login for the session, which is what you want here. Normal `claude` sessions are unaffected, because the variables are set only inside the launcher process.
- The three main aliases all resolve to `LMS_MAIN`, so `/model` cannot land on an unloaded model; `LMS_MAIN` is the one thing to change to compare models. Haiku is `LMS_SMALL`. Claude Code uses haiku for background work, and `CLAUDE_CODE_SUBAGENT_MODEL=haiku` would send subagents there too, but the launcher's tool set omits `Agent`, so no subagents run unless you override `--tools`. In practice the haiku model gets one two-message request per session — the title call — answered in about a second.
- `CLAUDE_CODE_TOTAL_TOKENS_REMINDER=off` stops Claude Code from appending a `role: "system"` message every turn (`<total_tokens>N tokens left</total_tokens>`, a budget note). Both LM Studio's Anthropic adapter and Ollama's qwen3.8 renderer fold mid-conversation system messages into the leading system turn, so each new one shifted every later token and destroyed the prefix cache on every request. Ollama's `ollama launch claude` sets this variable for the same reason (its v0.33.0 release notes); it is undocumented on the Claude Code side and was verified here against a fake upstream. `CLAUDE_CODE_ATTRIBUTION_HEADER=0` comes from the same place.
- `CLAUDE_CODE_MAX_CONTEXT_TOKENS=200000` tells Claude Code the window for an unrecognized model ID so it compacts in time, and `CLAUDE_CODE_DISABLE_1M_CONTEXT=1` makes that apply even when the account's default model alias carries a `[1m]` suffix. `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` compacts at 70% of the window. Compaction is itself a model request over the whole transcript — 62 s on the MoE in one session — so on a local model it should happen with headroom. The declared window is deliberately below the models' effective 262K and well above what Claude Code's estimator implies: with no `count_tokens` endpoint, Claude Code estimates from characters, and on tool-result-heavy content the estimate ran 1.94x the real count. At 200K and 70%, compaction fires at 140K estimated, roughly 70-105K real depending on content.

Flags the launcher adds unless you pass your own:

- `--effort medium`, as a flag rather than `CLAUDE_CODE_EFFORT_LEVEL`, because the env var is a hard override that `/effort` cannot change mid-session.
- `--permission-mode acceptEdits`. Auto mode cannot work locally: its classifier requests follow `ANTHROPIC_BASE_URL` and need a Claude model.
- `--tools Bash,Read,Edit,Write,Glob,Grep,AskUserQuestion,TodoWrite,TaskCreate,TaskGet,TaskList,TaskUpdate,EnterPlanMode,ExitPlanMode` — file and shell tools, the model's task list, and plan mode so it can propose before editing. No `Agent`, so no subagents; no `WebSearch`, which runs on Anthropic's servers. This matters for context: on a non-first-party base URL Claude Code sends every tool definition in full on every request, and the default built-in set is 21 tools and about 46 KB of JSON before `Artifact` and any connector tools. This set is about 12 KB. A local model executing a plan needs none of the rest. Unknown tool names are ignored, so the list is safe across versions.
- `--strict-mcp-config`, and a `--session-id` (skipped when resuming).

A trial that appended a short system prompt — read files in ranges, do not re-read after editing, prefer `git diff --stat` — cut a session's tool-result volume by about 40%. It was removed to keep the launcher plain; `--append-system-prompt` is the flag if you want that guidance back.

**Effort.** Claude Code sends `output_config: {"effort": ...}` and LM Studio maps it into the model's reasoning setting. Qwen3.8 has low/medium/xhigh levels and honors it. Qwen3.6 only has on/off, so any level becomes `on` — the log warns and falls back, which is harmless. `/effort` changes it mid-session.

**`/context` is an estimate, not a count.** LM Studio has no `count_tokens`, so Claude Code falls back to a character-based guess that runs about 1.3x above the real count on a fresh session and up to 2x once tool results dominate. Compaction triggers off the estimate, so it fires early rather than late, and the declared window is set with that in mind. One session compacted at 80,378 estimated against 41,455 real, after the model read a 48K-character file whole, edited it 16 times, read it whole again, and ran a full `git diff` on it. The model's habit of re-reading whole files is the biggest variable cost.

## Seeing what LM Studio is doing

`lms ps` for what is loaded, with the effective context and the parallel slot count (`--parallel`, default 4: how many predictions the model runs concurrently; idle slots cost nothing). `lms log stream` for live requests and responses; the files are under `~/.lmstudio/server-logs/`. Verbose logging is on by default and writes every request body to the log, which is useful for diagnosis and large.

`lms-session-stats [HH:MM] [session-id]` parses the log into a per-turn table: context, cached and new prompt tokens, prefill time, total time, and — given a Claude Code session ID — output tokens per turn and decode rate from the transcript. Per-turn alignment can be off by one; the totals are reliable. The `cached` column should climb with the conversation. If it sits flat while `new` stays large, the prefix cache is not holding.

`claude-local-sessions [dir]` lists a project's sessions newest first with the models each used and the first prompt; `claude-local --resume <id>` continues one.

Two log lines that are expected: `Unexpected endpoint or method. (HEAD /api/hello). Returning 200 anyway` is Claude Code's connection warm-up probe, logged at ERROR level but answered correctly. `Reasoning setting 'medium' is not supported by model 'qwen/qwen3.6-35b-a3b' ... falling back to 'on'` is the effort mapping on a model without levels.

One known LM Studio bug: `POST /v1/messages/count_tokens` returns HTTP 200 with an error body ([lmstudio-bug-tracker #2055](https://github.com/lmstudio-ai/lmstudio-bug-tracker/issues/2055)). Claude Code appears to retry that with growing backoff ("Waiting for API response, will retry"); it was seen once, in the session where every turn was also re-prefilling, and not since. Ollama returns 404, which Claude Code handles by falling back to an estimate. If the banner comes back, a pass-through that answers that one path with 404 is the workaround, pending the LM Studio fix.

## Codex as the client

Codex CLI speaks the OpenAI API, which LM Studio and `mlx_lm.server` both expose, so it can sit directly in front of either with no Anthropic translation. `~/.codex/config.toml` takes a custom provider with a `base_url` and, for servers that only implement chat completions, `wire_api = "chat"`. LM Studio serves both clients from one loaded model, so the cheap comparison is Claude Code and Codex against the same LM Studio instance on the same plan file.

## Ollama (kept for reference)

Ollama was the first backend. `ollama/modelfiles/` holds one Modelfile per model name the old launcher used; each pins `num_ctx` and, for the Qwen3.6 builds, Qwen's coding sampling (`temperature 0.6`, `presence_penalty 0`, which Ollama's plain tags do not set). `ollama create <name> -f ollama/modelfiles/<name>` bakes one into Ollama's store. `ollama/ollama-session-stats [logfile] [HH:MM]` parses `~/.ollama/logs/server.log` into the same per-turn table as `lms-session-stats`.

To use Ollama again: `ANTHROPIC_BASE_URL=http://localhost:11434`, `ANTHROPIC_AUTH_TOKEN=ollama`, the model names from the Modelfiles, and the same `CLAUDE_CODE_*` variables as the launcher. Ollama's own `ollama launch claude` sets the same environment, and maps haiku to the main model rather than to a separate small one — that works too, and is the fallback if Granite ever trips on the title call's JSON schema.

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
