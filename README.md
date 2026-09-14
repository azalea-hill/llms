# llms

Local-model Claude Code on Apple Silicon: LM Studio as the backend, three models, and a launcher. Goal is for every request to stay on the machine.

See [MODELS.md](MODELS.md) for model selection, memory usage, and per-machine recommendations.

```
bin/                       daily commands; put on PATH (see below)
  claude-local               launches Claude Code against LM Studio
  lms-load                   loads the main, sonnet, and haiku models
  lms-session-stats          per-session prefill, cache, and timing stats
  claude-local-sessions      lists a project's Claude Code sessions
ollama/                    archived Ollama modelfiles and session stats
MODELS.md                  model selection, memory, and per-machine recommendations
TODO.md                    todo list
```

## Setup

You need:

- **Apple Silicon Mac with 48 GB+ unified memory** See [MODELS.md](MODELS.md)
- **[LM Studio](https://lmstudio.ai)** (inference server)
- **[Claude Code](https://code.claude.com/docs/en/quickstart)** (client)
- **zsh** (macOS default; scripts use `#!/bin/zsh`)

### 1. Install Claude Code

```zsh
curl -fsSL https://claude.ai/install.sh | bash   # or: brew install --cask claude-code
claude --version
```

### 2. Install LM Studio and its CLI

Download from [lmstudio.ai](https://lmstudio.ai) and launch it once. The `lms` CLI ships inside it. If `lms` is not on your PATH:

```zsh
~/.lmstudio/bin/lms bootstrap
```

### 3. Download the three models

```zsh
lms get --mlx qwen/qwen3.6-35b-a3b        # main, MLX 4-bit, 20.4 GB
lms get --mlx google/gemma-4-26b-a4b-qat  # sonnet, MLX 4-bit, 15 GB
lms get ibm/granite-4-micro               # haiku, GGUF, 2.1 GB
lms ls
```

- `qwen/qwen3.6-35b-a3b` answers as fable and opus
- `google/gemma-4-26b-a4b-qat` answers as sonnet (the auto-mode classifier)
- `ibm/granite-4-micro` answers as haiku (session titles, `WebFetch` summaries, `--resume` summaries)

Both are kept off the main model so they cannot evict its prompt cache, and haiku gets a model with no thinking mode because Claude Code sends those calls at `effort: high`. See [MODELS.md](MODELS.md) for why these three.

### 4. Put `bin/` on your PATH

Either symlink the scripts in ~/.local/bin/ or add the bin directory to `~/.zshrc`:

```zsh
export PATH="$HOME/apps/ah/llms/bin:$PATH"
```

### 5. Run

```zsh
lms server start    # once per boot; listens on :1234
lms-load            # loads the three; ~45 s
claude-local        # from whatever repo you want to work in
```

First turn of a session will be slow (system prompt + tool definitions). `lms ps` shows what is resident; `lms unload --all` frees it.

## Loading models: `lms-load`

```zsh
lms-load                              # defaults
lms-load qwen/qwen3.8-27b             # different main model
LMS_MEDIUM=qwen/qwen3.6-35b-a3b lms-load  # same model for sonnet
LMS_SMALL=qwen2.5-14b-instruct-mlx lms-load  # different haiku model
```

Unloads everything, then loads all three models at 200K context with `--gpu max`. Only one main model is resident at a time; switching models requires another `lms-load`.

## The launcher: `claude-local`

```zsh
claude-local
LMS_MAIN=qwen/qwen3.8-27b claude-local
LMS_MEDIUM=qwen/qwen3.6-35b-a3b claude-local
LMS_SMALL=google/gemma-4-26b-a4b-qat claude-local
claude-local --resume <id>
```

See [MODELS.md](MODELS.md) for environment variable details, tool flags, and why the prompt cache settings are configured as they are.

## Monitoring

- `lms ps` — what is loaded, effective context, parallel slots
- `lms log stream` — live requests and responses (under `~/.lmstudio/server-logs/`)
- `lms-session-stats [HH:MM] [session-id]` — per-turn table: context, cached/new tokens, prefill time, decode rate
- `claude-local-sessions [dir]` — lists a project's sessions

## Reference

- [Claude Code model config](https://code.claude.com/docs/en/model-config)
- [Claude Code env vars](https://code.claude.com/docs/en/env-vars)
- [Claude Code gateway protocol](https://code.claude.com/docs/en/llm-gateway-protocol)
- [LM Studio Claude Code integration](https://lmstudio.ai/blog/claudecode)
- [LM Studio CLI](https://lmstudio.ai/docs/cli)
