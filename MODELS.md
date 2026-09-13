# Model selection

What to run and what it costs in memory. Setup and the launcher live in [README.md](README.md).

**`qwen/qwen3.6-35b-a3b` at MLX 4-bit as the main model, `ibm/granite-4-micro` as haiku, about 31 GB resident.**

## The two roles

The **main model** answers as fable, opus, and sonnet — every turn you see. It holds a long transcript, calls tools, and generates thousands of tokens while you wait, so decode speed matters as much as quality.

The **haiku model** handles background calls: session titles, summaries, and subagents if you enable `Agent`. Roughly one short request per session.

Haiku has one hard constraint: Claude Code's session-title call hardcodes `effort: high`, and LM Studio maps request-level effort into the model's reasoning setting, where the request wins over the model's default. A haiku model with a thinking mode reasons at length about a three-word title — Gemma 4 12B did this. Granite has no reasoning mode, so there is nothing to turn on. That plus 2.1 GB is the case for it.

## Dense vs mixture-of-experts

A **dense** model runs every weight for every token. A **mixture-of-experts (MoE)** model routes each token through a fraction of its parameters: `qwen3.6-35b-a3b` is 35B total, ~3B active per token.

This is the axis that matters on Apple Silicon, where generation is bound by memory bandwidth, not compute. Each token streams the active weights through the GPU once, so decode speed ≈ bandwidth ÷ active bytes. On the 64 GB M5 Pro (307 GB/s), the dense 27B at 4-bit reads ~15 GB per token and measured 15-16 tok/s; the 3B-active MoE reads ~2 GB and measured 71-74 tok/s.

The full 35B still has to be resident — routing is per token and unpredictable. **You pay dense memory for sparse speed**, which is the right trade for an agent generating thousands of tokens per session.

## 4-bit vs 8-bit

Quantization stores each weight in fewer bits than the 16-bit floats the model was trained in: 8-bit is about half the original size, 4-bit about a quarter. Because decode is bandwidth-bound, fewer bits means proportionally faster decode. The cost is precision.

Measured on the MoE: 4-bit is 20.4 GB and 71-74 tok/s; 8-bit is ~37 GB and 44-53 tok/s. 4-bit is the default here. 8-bit is worth considering once the machine has the memory to spare.

Naming differs by runtime — MLX uses `4bit`/`8bit`, GGUF uses `q4_k_m`-style names, Ollama's `-nvfp4` and `-mxfp8` tags are its 4-bit and 8-bit MLX builds. Switching variants has to be done in LM Studio's UI, not the CLI; see [README.md](README.md#loading-models-lms-load).

## Models tried

| Model                        | Type                       | Role  | Outcome                                                |
| ---------------------------- | -------------------------- | ----- | ------------------------------------------------------ |
| `qwen/qwen3.6-35b-a3b` 4-bit | MoE, 35B total / 3B active | main  | **Default.** 71-74 tok/s decode.                       |
| `qwen/qwen3.6-35b-a3b` 8-bit | same                       | main  | 44-53 tok/s, ~37 GB weights.                           |
| `qwen/qwen3.8-27b` 4-bit     | dense 27B                  | main  | 15-16 tok/s. Too slow to sit through at 307 GB/s.      |
| `qwen/qwen3.8-27b` 8-bit     | dense 27B                  | main  | 15-20 tok/s.                                           |
| `gemma4:26b-mlx` (26B-A4B)   | MoE                        | main  | Tried on Ollama, not carried forward.                  |
| `gpt-oss:20b`                | MoE                        | main  | Tried on Ollama, not carried forward.                  |
| `ibm/granite-4-micro`        | dense 3B                   | haiku | **Default.** No reasoning mode. 2.1 GB.                |
| `gemma4:12b-mlx`             | dense 12B                  | haiku | Thinking mode; reasons at length about session titles. |
| `qwen3.5:4b`                 | dense 4B                   | -     | Too small.                                             |

No systematic quality comparison was run. The numbers above are throughput only.

## Measurements

A PR-summarization prompt against a real repo, about 9K real first-turn tokens with the trimmed tool set, 2026-09-11/12, **on the 64 GB M5 Pro (307 GB/s)**. Time is first user message to final answer.

|                       | Opus 5      | qwen3.6 4-bit, LM Studio | qwen3.6 8-bit, LM Studio | qwen3.6 4-bit, Ollama | qwen3.8 4-bit, LM Studio | qwen3.8 4-bit, Ollama |
| --------------------- | ----------- | ------------------------ | ------------------------ | --------------------- | ------------------------ | --------------------- |
| Time to answer        | 45 s        | 59 s                     | 46 s                     | 104 s                 | 458 s                    | 699 s                 |
| Requests / tool calls | 4 / 3       | 7 / 12                   | 3 / 6                    | 10 / 18               | 9 / 11                   | 8 / 14                |
| First-turn prefill    | n/a         | 1,924 tok/s              | 1,594 tok/s              | 978 tok/s             | 347 tok/s                | 366 tok/s             |
| Decode                | n/a         | ~71-74 tok/s             | 44-53 tok/s              | 48-85 tok/s           | ~15-16 tok/s             | 24-40 tok/s           |
| Tokens prefilled      | n/a         | 24K                      | 19K                      | 26K                   | 26.5K                    | 175K (cache broken)   |
| Output tokens         | 4,382       | 3,194                    | 1,802                    | 4,476                 | 5,410                    | 6,906                 |
| Final answer          | 5,531 chars | 6,260                    | 4,434                    | 7,416                 | 5,806                    | 6,627                 |

One run per configuration, and the models vary their tool-call plans between runs, so wall-clock and tool-call counts are single samples. Prefill and decode rates are the stable signals. The Ollama qwen3.8 row predates `CLAUDE_CODE_TOTAL_TOKENS_REMINDER=off`; its 175K of re-prefill was the system-message folding problem, not the runtime.

Simple-prompt decode, nothing else running, on Ollama: 3.6 MoE 4-bit ~89 tok/s, 8-bit ~81; 3.8 dense 27B 4-bit 34 tok/s, 8-bit ~15-20. The dense model is bandwidth-bound so 8-bit roughly halves it; the MoE barely moves.

Bandwidth beats capacity across machines: a 48 GB M4 Max (546 GB/s) ran the 4-bit MoE at ~116 tok/s against ~89 on the 64 GB M5 Pro (307 GB/s). Prefill goes the other way — without the M5's neural accelerators the M4 Max prefills two to three times slower.

## What the set costs in memory

From `lms load --estimate-only` on a 48 GB M4 Max:

| What                             | Context target           | Estimated working set |
| -------------------------------- | ------------------------ | --------------------- |
| `qwen/qwen3.6-35b-a3b` MLX 4-bit | 32K / 131K / 200K / 262K | **26.6 GB at each**   |
| `ibm/granite-4-micro` GGUF       | 32K                      | **4.4 GB**            |
| The pair, as `lms-load` loads it | 200K + 32K               | **~31 GB**            |

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

"Free" below is physical memory minus the resident model set — what is left for macOS and everything else you run.

| Machine    | Metal ceiling | Model set                                   | Resident | Free   |
| ---------- | ------------- | ------------------------------------------- | -------- | ------ |
| **48 GB**  | ~36 GB        | qwen3.6-35b-a3b 4-bit + Granite             | ~31 GB   | ~17 GB |
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
