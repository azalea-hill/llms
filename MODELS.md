# Model selection

What to run and what it costs in memory. Setup and the launcher live in [README.md](README.md).

**`qwen/qwen3.6-35b-a3b` at MLX 4-bit as the main model, `google/gemma-4-26b-a4b-qat` as sonnet, `ibm/granite-4-micro` as haiku, about 54 GB resident, with the two MLX figures lazy ceilings.**

## The roles

The **main model** answers as fable and opus — every turn you see. It holds a long transcript, calls tools, and generates thousands of tokens while you wait, so decode speed matters as much as quality.

The **sonnet model** answers the auto-mode permission classifier: one or more ~32K-token requests per Bash call while auto mode is on, none otherwise. The prompt is mostly constant, so after one cold prefill the calls hit the model's own cache. Gemma 4 26B-A4B is the pick because it is a 4B-active MoE (fast prefill, 15 GB) and the classifier task is tag-formatted judgment rather than coding. Granite 4 Micro was tried first and could not do it: it screened every action to stage 2, needed one to four retries per verdict to emit parseable tags, blocked a read-only `find` with a reason about an earlier `git stash`, and allowed `ls` on reasoning that was not about `ls`. Gemma's verdict quality against Sonnet's is unmeasured.

The **haiku model** handles background calls: session titles, `WebFetch` page summaries, `--resume` summaries, prompt suggestions, and subagents if you enable `Agent`. Claude Code sends every one of them at `effort: high`, and LM Studio maps request-level effort into the model's reasoning setting, where the request wins over the model's default — so a haiku model with a thinking mode reasons at length about a three-word title. Gemma 26B-A4B was tried in the slot: a 9K-token `WebFetch` summary took 56-67 s (8 s of prefill, the rest thinking) and one session title took 77 s, all while competing with the main model for the GPU. So the slot needs a model with no thinking mode, which rules out the current generation (Qwen3.x, Gemma 4, gpt-oss). Granite 4 Micro (3B, GGUF, 2.1 GB) qualifies and answers in a few seconds. Its `WebFetch` summaries are thin, but that is the tool, not the model: the extractor only answers the prompt the main model passes, over a page Claude Code has already truncated, and Qwen2.5-14B-Instruct (pre-thinking, 8.3 GB, fetched by Hugging Face URL since the catalog does not list it) produced no fuller summaries than Granite. `WebFetch` is for triage; when a page matters, the main model should `curl` it through Bash and read the whole thing. Granite's GGUF context is preallocated (4.4 GB at 32K, 6.7 GB at 64K, 16 GB at 200K) and `lms-load` gives it 64K, enough for truncated pages and titles; only a `--resume` summary of a very long session could approach it, where it would truncate rather than compact since Claude Code has no per-tier context setting. The alternative that keeps one background model is a local virtual model (`model.yaml` with Gemma as `base` and `metadataOverrides.reasoning: false`, which is how the hub's Granite definition disables reasoning); untried.

## Dense vs mixture-of-experts

A **dense** model runs every weight for every token. A **mixture-of-experts (MoE)** model routes each token through a fraction of its parameters: `qwen3.6-35b-a3b` is 35B total, ~3B active per token.

This is the axis that matters on Apple Silicon, where generation is bound by memory bandwidth, not compute. Each token streams the active weights through the GPU once, so decode speed ≈ bandwidth ÷ active bytes. On the 64 GB M5 Pro (307 GB/s), the dense 27B at 4-bit reads ~15 GB per token and measured 15-16 tok/s; the 3B-active MoE reads ~2 GB and measured 71-74 tok/s.

The full 35B still has to be resident — routing is per token and unpredictable. **You pay dense memory for sparse speed**, which is the right trade for an agent generating thousands of tokens per session.

## 4-bit vs 8-bit

Quantization stores each weight in fewer bits than the 16-bit floats the model was trained in: 8-bit is about half the original size, 4-bit about a quarter. Because decode is bandwidth-bound, fewer bits means proportionally faster decode. The cost is precision.

Measured on the MoE: 4-bit is 20.4 GB and 71-74 tok/s; 8-bit is ~37 GB and 44-53 tok/s. 4-bit is the default here. 8-bit is worth considering once the machine has the memory to spare.

Naming differs by format — MLX uses `4bit`/`8bit`, GGUF uses `q4_k_m`-style names. Switching variants has to be done in LM Studio's UI, not the CLI; see [README.md](README.md#loading-models-lms-load).

## Models tried

| Model                        | Type                       | Role          | Outcome                                                                      |
| ---------------------------- | -------------------------- | ------------- | ---------------------------------------------------------------------------- |
| `qwen/qwen3.6-35b-a3b` 4-bit | MoE, 35B total / 3B active | main          | **Default.** 71-74 tok/s decode.                                             |
| `qwen/qwen3.6-35b-a3b` 8-bit | same                       | main          | 44-53 tok/s, ~37 GB weights.                                                 |
| `qwen/qwen3.8-27b` 4-bit     | dense 27B                  | main          | 15-16 tok/s. Too slow to sit through at 307 GB/s.                            |
| `qwen/qwen3.8-27b` 8-bit     | dense 27B                  | main          | 15-20 tok/s.                                                                 |
| `google/gemma-4-26b-a4b-qat` | MoE                        | sonnet        | **Default.** Classifier. 15 GB. Thinks on haiku calls.                       |
| gpt-oss 20B                  | MoE                        | main          | Tried, not carried forward.                                                  |
| `ibm/granite-4-micro`        | dense 3B                   | haiku         | **Default.** No thinking mode. 2.1 GB.                                       |
| `qwen2.5-14b-instruct-mlx`   | dense 14B                  | haiku         | No thinking mode; `WebFetch` summaries no fuller than Granite's. 8.3 GB.     |
| Gemma 4 12B                  | dense 12B                  | haiku         | Thinking mode; reasons at length about session titles.                       |
| Qwen3.5 4B                   | dense 4B                   | -             | Too small.                                                                   |

No systematic quality comparison was run. The numbers above are throughput only.

## Measurements

A PR-summarization prompt against a real repo, about 9K real first-turn tokens, **on the 64 GB M5 Pro (307 GB/s)**. Time is first user message to final answer.

|                       | Opus 5      | qwen3.6 4-bit | qwen3.6 8-bit | qwen3.8 4-bit |
| --------------------- | ----------- | ------------- | ------------- | ------------- |
| Time to answer        | 45 s        | 59 s          | 46 s          | 458 s         |
| Requests / tool calls | 4 / 3       | 7 / 12        | 3 / 6         | 9 / 11        |
| First-turn prefill    | n/a         | 1,924 tok/s   | 1,594 tok/s   | 347 tok/s     |
| Decode                | n/a         | ~71-74 tok/s  | 44-53 tok/s   | ~15-16 tok/s  |
| Tokens prefilled      | n/a         | 24K           | 19K           | 26.5K         |
| Output tokens         | 4,382       | 3,194         | 1,802         | 5,410         |
| Final answer          | 5,531 chars | 6,260         | 4,434         | 5,806         |

One run per configuration, and the models vary their tool-call plans between runs, so wall-clock and tool-call counts are single samples. Prefill and decode rates are the stable signals.

Simple-prompt decode, nothing else running: 3.6 MoE 4-bit ~89 tok/s, 8-bit ~81; 3.8 dense 27B 4-bit 34 tok/s, 8-bit ~15-20. The dense model is bandwidth-bound so 8-bit roughly halves it; the MoE barely moves.

Bandwidth beats capacity across machines: a 48 GB M4 Max (546 GB/s) ran the 4-bit MoE at ~116 tok/s against ~89 on the 64 GB M5 Pro (307 GB/s). Prefill goes the other way — without the M5's neural accelerators the M4 Max prefills two to three times slower.

## What the set costs in memory

From `lms load --estimate-only` on a 48 GB M4 Max:

| What                                   | Context target           | Estimated working set |
| -------------------------------------- | ------------------------ | --------------------- |
| `qwen/qwen3.6-35b-a3b` MLX 4-bit       | 32K / 131K / 200K / 262K | **26.6 GB at each**   |
| `google/gemma-4-26b-a4b-qat` MLX 4-bit | 32K / 200K               | **20.4 GB at each**   |
| `ibm/granite-4-micro` GGUF             | 32K / 64K / 200K         | **4.4 / 6.7 / 16.4 GB** |
| The three, as `lms-load` loads them    | 200K + 200K + 64K        | **~54 GB**            |

The MLX estimate does not move with context because MLX allocates KV lazily — 26.6 GB is the ceiling, and real usage climbs toward it as the transcript fills. The GGUF estimate does move, because llama.cpp preallocates; Granite's 4.4 GB is mostly cache for a 2.1 GB model, so lowering its context target is the cheapest memory to recover.

The 8-bit MoE was not estimated here. Doubling the weights and keeping the overhead puts the pair near 49 GB — extrapolation, not measurement.

Disk is 22.6 GB for both models.

## macOS and the GPU memory ceiling

macOS caps how much unified memory Metal will wire. `sysctl iogpu.wired_limit_mb` returns `0` on a stock machine, meaning the default: **about 75% of physical memory** above 36 GB. Weights loaded for Metal are wired and never page out, so the ceiling is a hard wall.

Raise it until the next reboot:

```zsh
sudo sysctl iogpu.wired_limit_mb=40960    # 40 GB on a 48 GB Mac
```

Do not raise it while you have a heavy working set — you are taking memory from the applications, and macOS will swap them rather than refuse the GPU. The 75% figure is a default rather than a guarantee; Metal reports the real value as `recommendedMaxWorkingSetSize`, and it measures slightly higher on some machines.

## Recommendations by machine

"Free" below is physical memory minus the resident model set — what is left for macOS and everything else you run. Rows without Gemma are the layout with `LMS_MEDIUM=$LMS_MAIN`, where the sonnet slot shares the main model and auto mode is off or accepted as slow; the three-model row is the default, and its ~54 GB is a ceiling — the two MLX models allocate KV lazily, so real usage starts near 40 GB and grows with the transcripts — but it leaves no room for Docker on 64 GB.

| Machine    | Metal ceiling | Model set                                   | Resident | Free   |
| ---------- | ------------- | ------------------------------------------- | -------- | ------ |
| **48 GB**  | ~36 GB        | qwen3.6-35b-a3b 4-bit + Granite             | ~31 GB   | ~17 GB |
| **64 GB**  | ~48 GB        | qwen3.6-35b-a3b 4-bit + Gemma + Granite     | ~54 GB   | ~10 GB |
| **64 GB**  | ~48 GB        | qwen3.6-35b-a3b 4-bit + Granite             | ~31 GB   | ~33 GB |
| **64 GB**  | ~48 GB        | qwen3.6-35b-a3b 8-bit + Granite             | ~49 GB   | ~15 GB |
| **128 GB** | ~96 GB        | gpt-oss-120b + Granite                      | ~65 GB   | ~63 GB |
| **128 GB** | ~96 GB        | Qwen3.5 122B-A10B 4-bit + Granite           | ~85 GB   | ~43 GB |
| **256 GB** | ~192 GB       | DeepSeek-V4-Flash 284B-A13B 4-bit + Granite | ~160 GB  | ~96 GB |

The 8-bit row on 64 GB needs the ceiling raised to ~56 GB and leaves little for anything else. Everything at 128 GB and above is from published figures, not measured here.

**What the bigger models buy you.** More total parameters at the same active count is close to free speed-wise, so on a large machine the upgrade path is a wider MoE rather than a dense model or a higher quantization:

- **[gpt-oss-120b](https://huggingface.co/openai/gpt-oss-120b)** — 117B total, 5.1B active, ships natively in MXFP4 at [~60 GB](https://aliteq.com/gpt-oss-120b-hardware-requirements-2026). The best quality-per-byte step up from the 35B, and only slightly more active weight per token, so it stays quick.
- **Qwen3.5 122B-A10B** — ~81 GB at 4-bit, 262K context. 10B active means roughly a third the decode rate of a 3B-active model at the same bandwidth; better for planning than for watching it edit.
- **Qwen3-Coder-Next 80B-A3B** — tuned for agentic coding, 3B active, 262K context, [>45 GB at 4-bit](https://huggingface.co/unsloth/Qwen3-Coder-Next-GGUF). Fits a 64 GB machine with the ceiling raised, and is the most interesting untested candidate for this setup.
- **DeepSeek-V4-Flash** — 284B total, ~13B active, 1M context, ~155 GB at 4-bit or ~90 GB at 2-bit. Measured around 35 tok/s on a 512 GB M3 Ultra. The 2-bit build fits 128 GB at reduced quality.
- **GLM-5.3** is the strongest open-weight coding model as of late 2026 but needs roughly 418 GB, so it is out of reach below a 512 GB machine.

On 128 GB and up you can keep two main models resident under separate `--identifier`s: a wide MoE for planning and the fast 35B for execution, which is the [workflow in README.md](README.md#workflow) without the reload between steps.

Below 48 GB this pair does not fit — the MoE alone is 26.6 GB against a ~24 GB ceiling on a 32 GB Mac. A smaller MoE is the realistic option there.

## Launcher configuration

### Environment variables

```
ANTHROPIC_BASE_URL=http://localhost:1234
ANTHROPIC_AUTH_TOKEN=lmstudio
ANTHROPIC_API_KEY=
ANTHROPIC_MODEL=opus
ANTHROPIC_DEFAULT_FABLE_MODEL=$LMS_MAIN
ANTHROPIC_DEFAULT_OPUS_MODEL=$LMS_MAIN
ANTHROPIC_DEFAULT_SONNET_MODEL=$LMS_MEDIUM
ANTHROPIC_DEFAULT_HAIKU_MODEL=$LMS_SMALL
CLAUDE_CODE_SUBAGENT_MODEL=haiku
CLAUDE_CODE_TOTAL_TOKENS_REMINDER=off
CLAUDE_CODE_ATTRIBUTION_HEADER=0
CLAUDE_CODE_MAX_CONTEXT_TOKENS=200000
CLAUDE_CODE_DISABLE_1M_CONTEXT=1
CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70
CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
```

- `ANTHROPIC_BASE_URL` sends all inference to LM Studio (override with `LMS_URL`). `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` (officially documented) turns off telemetry, error reporting, feature-flag fetches, and the update check. Model discovery still runs through the gateway (since v2.1.257), so a minimal request may reach Anthropic at startup.
- `ANTHROPIC_AUTH_TOKEN` with an empty `ANTHROPIC_API_KEY` satisfies Claude Code's credential check; LM Studio ignores the value unless Require Authentication is on.
- The launcher sets three tiers: fable/opus is `LMS_MAIN`, sonnet is `LMS_MEDIUM`, haiku is `LMS_SMALL` (default `ibm/granite-4-micro`). Use `LMS_MEDIUM=$LMS_MAIN` for sonnet on the main model; `LMS_SMALL=$LMS_MAIN` for haiku.
- `CLAUDE_CODE_TOTAL_TOKENS_REMINDER=off` stops Claude Code from appending a `<total_tokens>` budget note every turn. LM Studio's Anthropic adapter folds mid-conversation system messages into the leading system turn, so each new one shifted every later token and destroyed the prefix cache.
- `CLAUDE_CODE_ATTRIBUTION_HEADER=0` (officially documented) omits the system prompt attribution block. Since v2.1.229, classifier requests keep the block when requests go through `api.anthropic.com` with non-profile credentials, so this only affects non-classifier requests.
- `CLAUDE_CODE_MAX_CONTEXT_TOKENS=200000` tells Claude Code the window for an unrecognized model ID so it compacts in time. `CLAUDE_CODE_DISABLE_1M_CONTEXT=1` makes that apply even when the account's default model alias carries a `[1m]` suffix. `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` compacts at 70% of the window.

### Flags

Unless you pass your own, the launcher adds:

- `--effort medium` — as a flag rather than env var (the env var is a hard override that `/effort` cannot change mid-session).
- `--permission-mode acceptEdits` — auto mode's classifier requests follow `ANTHROPIC_BASE_URL` to the sonnet slot, but its verdict quality against Sonnet's is unmeasured.
- `--tools Bash,Read,Edit,Write,Glob,Grep,WebFetch,AskUserQuestion,TodoWrite,TaskCreate,TaskGet,TaskList,TaskUpdate,EnterPlanMode,ExitPlanMode` — file and shell tools, `WebFetch`, task list, and plan mode. No `Agent` (no subagents), no `WebSearch` (executed by Anthropic's API). `WebFetch` fetches the page itself and summarizes it with a haiku-tier call. The launcher's `--settings` disables Anthropic's hostname blocklist check (`skipWebFetchPreflight`) and adds a `permissions.ask` rule for `WebFetch` so every fetch prompts. This set is ~12 KB; the default built-in is 21 tools and ~46 KB.
- `--strict-mcp-config` and `--settings '{"skipWebFetchPreflight":true,"permissions":{"ask":["WebFetch"]}}'`
- `--session-id` (skipped when resuming)

### Context and compaction

LM Studio has no `count_tokens`, so Claude Code falls back to a character-based guess that runs about 1.3x above the real count on a fresh session and up to 2x once tool results dominate. Compaction triggers off the estimate, so it fires early rather than late. The declared 200K window is deliberately below the models' effective 262K. One session compacted at 80,378 estimated against 41,455 real, after the model read a 48K-character file whole, edited it 16 times, read it whole again, and ran a full `git diff` on it.
