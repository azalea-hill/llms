# Local Claude Code with LM Studio

This setup routes Claude Code model inference through three models in LM Studio. Qwen handles the main coding work, while separate models handle Claude Code's Sonnet and Haiku request slots.

See the shared [model and hardware notes](../docs/models.md) for model selection, memory usage, and per-machine recommendations.

```text
../bin/                      Shared command directory; put on PATH in the root setup.
  claude-local               Launch Claude Code against LM Studio.
  lms-load                   Load the main, Sonnet, and Haiku models.
  lms-session-stats          Show per-session prefill, cache, and timing statistics.
  claude-local-sessions      List a project's Claude Code sessions.
../ollama/                   Archived Ollama experiments; not used by this setup.
../docs/models.md            Model selection, memory usage, and per-machine recommendations.
todo.md                      Claude-specific backlog.
```

## Setup

First complete the repository's [shared LM Studio setup](../README.md#shared-setup). The default three-model configuration targets a 64 GB Apple Silicon Mac and can grow toward a roughly 54 GB working set. The [model and hardware notes](../docs/models.md) describe a possible two-model layout for a 48 GB Mac, but the current `lms-load` command does not automate that layout.

### 1. Install Claude Code

```zsh
curl -fsSL https://claude.ai/install.sh | bash
claude --version
```

The [Claude Code quickstart](https://code.claude.com/docs/en/quickstart) also documents other installation methods.

### 2. Download the routing models

The shared setup already downloaded Qwen. Add the two models used for Claude Code's smaller request slots:

```zsh
lms get --mlx google/gemma-4-26b-a4b-qat
lms get ibm/granite-4-micro
lms ls
```

- `qwen/qwen3.6-35b-a3b` answers as Fable and Opus.
- `google/gemma-4-26b-a4b-qat` answers as Sonnet, including auto-mode classification.
- `ibm/granite-4-micro` answers as Haiku for session titles, `WebFetch` summaries, and `--resume` summaries.

The two routing models stay separate from the main model so their requests do not evict its prompt cache. Granite has no thinking mode, which avoids wasting work when Claude Code sends Haiku-slot calls with `effort: high`. See the [model and hardware notes](../docs/models.md) for the measured rationale.

### 3. Load the models

```zsh
lms-load
```

The loader unloads currently resident models, then loads all three configured models. Allow roughly 45 seconds on the reference machine.

### 4. Launch Claude Code

From the repository you want Claude Code to work on:

```zsh
claude-local
```

The first turn of a session is slower because LM Studio must process the system prompt and tool definitions. Use `lms ps` to see what is resident and `lms unload --all` to free it.

## Loading different models

```zsh
lms-load
lms-load qwen/qwen3.8-27b
LMS_MEDIUM=qwen/qwen3.6-35b-a3b lms-load
LMS_SMALL=qwen2.5-14b-instruct-mlx lms-load
```

`lms-load` uses a 200K context for the main and medium models, a 65,536-token context for the small model, and maximum GPU offload. Running it unloads every resident model first.

## Launcher options

```zsh
claude-local
LMS_MAIN=qwen/qwen3.8-27b claude-local
LMS_MEDIUM=qwen/qwen3.6-35b-a3b claude-local
LMS_SMALL=google/gemma-4-26b-a4b-qat claude-local
claude-local --resume SESSION_ID
```

See the [model and hardware notes](../docs/models.md) for environment-variable details, tool flags, and prompt-cache settings.

## Monitoring

- `lms ps` shows loaded models, effective context, and parallel slots.
- `lms log stream` streams live requests and responses from `~/.lmstudio/server-logs/`.
- `lms-session-stats [HH:MM] [session-id]` shows per-turn context, cached and new tokens, prefill time, and decode rate.
- `claude-local-sessions [directory]` lists a project's sessions.

## Configuration examples

- [config/claude.md.example](config/claude.md.example) is a snapshot of the current global `~/.claude/CLAUDE.md` working agreements. It is the single source for both tools; `~/.codex/AGENTS.md` is a symlink to it.
- [config/settings.json.example](config/settings.json.example) is a snapshot of the current global `~/.claude/settings.json` minus terminal cosmetics. The `permissions.deny` list mirrors the Codex [default.rules](../codex/config/default.rules.example) prefix rules, plus `EnterWorktree` and `ExitWorktree` because those native tools create git worktrees without going through Bash. `includeGitInstructions: false` drops the built-in commit and PR workflow from the system prompt, since the agent never commits.

The `.example` suffix keeps these from being loaded as active configuration when this repository is opened. Review and merge only the settings you want; the repository does not install user configuration.

## Repository-local Claude settings

This repository does not provide a `.claude/settings.local.json`. If you add one for this checkout, keep it at the repository root because Claude Code discovers project-local settings there when launched from this directory. Review any permissions in that file separately from the launcher configuration.

## References

- [Claude Code quickstart](https://code.claude.com/docs/en/quickstart)
- [Claude Code model configuration](https://code.claude.com/docs/en/model-config)
- [Claude Code environment variables](https://code.claude.com/docs/en/env-vars)
- [Claude Code gateway protocol](https://code.claude.com/docs/en/llm-gateway-protocol)
- [LM Studio Claude Code integration](https://lmstudio.ai/blog/claudecode)
