# llms

Local-model coding-agent setups for Apple Silicon, with LM Studio as the inference backend. The repository supports [Claude Code](claude/README.md) and [Codex](codex/README.md) while keeping their client-specific configuration separate.

## Shared setup

Complete these steps before following either client guide. The shared Qwen configuration expects at least 48 GB of unified memory. The default Claude configuration targets a 64 GB Mac and can grow toward a roughly 54 GB working set; see the [model and hardware notes](docs/models.md) before downloading its additional models.

### 1. Install LM Studio

Download and install [LM Studio](https://lmstudio.ai/download), then launch the application once. The `lms` CLI ships with LM Studio.

Verify the CLI:

```zsh
~/.lmstudio/bin/lms --help
```

### 2. Put the CLIs on your PATH

From this repository's root, add LM Studio and the repository scripts to the current shell:

```zsh
export PATH="$HOME/.lmstudio/bin:$PWD/bin:$PATH"
```

Add the equivalent line with this repository's absolute path to `~/.zshrc` if you want the commands in future shells. Verify the shared and client launchers are visible:

```zsh
command -v lms lms-load claude-local lms-load-codex codex-local
```

Alternatively, link only the commands you use into a directory already on `PATH`. The repository does not create or overwrite these links:

```zsh
mkdir -p "$HOME/.local/bin"
ln -s "$PWD/bin/lms-load" "$HOME/.local/bin/lms-load"
ln -s "$PWD/bin/claude-local" "$HOME/.local/bin/claude-local"
ln -s "$PWD/bin/lms-session-stats" "$HOME/.local/bin/lms-session-stats"
ln -s "$PWD/bin/claude-local-sessions" "$HOME/.local/bin/claude-local-sessions"
ln -s "$PWD/bin/lms-load-codex" "$HOME/.local/bin/lms-load-codex"
ln -s "$PWD/bin/codex-local" "$HOME/.local/bin/codex-local"
ln -s "$PWD/bin/codex-model-catalog" "$HOME/.local/bin/codex-model-catalog"
ln -s "$PWD/bin/codex-export-models" "$HOME/.local/bin/codex-export-models"
ln -s "$PWD/bin/codex-export-prompts" "$HOME/.local/bin/codex-export-prompts"
```

### 3. Download the shared Qwen model

Both configurations use Qwen as their main coding model:

```zsh
lms get --mlx qwen/qwen3.6-35b-a3b
lms ls
```

LM Studio may ask you to choose a quantization. The scripts and recorded memory figures use the MLX 4-bit variant.

### 4. Start the local server

Both launchers expect LM Studio's OpenAI-compatible server at `http://localhost:1234`:

```zsh
lms server start --port 1234
lms server status
```

Keep the default localhost bind unless you intentionally want other machines to reach the server. Binding it to another interface expands access beyond this Mac.

### 5. Finish the client-specific setup

- [Set up Claude Code](claude/README.md#setup)
- [Set up Codex](codex/README.md#setup)

## Commands

| Command                 | Purpose                                        |
| ----------------------- | ---------------------------------------------- |
| `lms-load`              | Load the three models used by Claude Code.     |
| `claude-local`          | Launch Claude Code against LM Studio.          |
| `lms-session-stats`     | Show LM Studio timing and cache statistics.    |
| `claude-local-sessions` | List Claude Code sessions for a project.       |
| `lms-load-codex`        | Load the model used by local Codex.            |
| `codex-local`           | Launch Codex against LM Studio.                |
| `codex-model-catalog`   | Generate the custom local Codex model catalog. |
| `codex-export-models`   | Snapshot Codex's bundled OpenAI model catalog. |
| `codex-export-prompts`  | Extract the bundled GPT-5.6 Sol base prompt.   |

## Documentation

- [Hybrid local Codex implementation plan](docs/codex-hybrid-plan.md)
- [Model and hardware notes](docs/models.md)
- [Cross-cutting backlog](docs/todo.md)
- [Claude-specific backlog](claude/todo.md)
- [Codex model-catalog notes](codex/models/model-catalog.md)
- [Codex tool surface](codex/tools.md)
- [Local Codex `apply_patch` compatibility](codex/apply-patch.md)
- [Archived Ollama experiments](ollama/README.md)

## Shared references

- [Download LM Studio](https://lmstudio.ai/download)
- [LM Studio CLI](https://lmstudio.ai/docs/cli)
- [Download models with `lms get`](https://lmstudio.ai/docs/cli/local-models/get)
- [Start the LM Studio server](https://lmstudio.ai/docs/cli/serve/server-start)
