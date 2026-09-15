# Local Codex with LM Studio

This setup sends ordinary Codex model requests to `qwen/qwen3.6-35b-a3b` through LM Studio. Native Codex web search is disabled; an explicit OpenAI planning workflow and narrow web tooling remain later phases of the [implementation plan](../docs/codex-hybrid-plan.md). The routing keeps routine inference local without trying to make Codex completely isolated from OpenAI.

## Setup

First complete the repository's [shared LM Studio setup](../README.md#shared-setup), including downloading Qwen and starting the server on port `1234`.

### 1. Install the Codex CLI

Install or update Codex on macOS:

```zsh
curl -fsSL https://chatgpt.com/codex/install.sh | sh
codex --version
```

This repository's scripts and local instructions also require Bash, `curl`, `jq`, and `rg` from ripgrep:

```zsh
command -v bash curl jq rg
```

If `jq` or `rg` is missing and you use Homebrew, install them with `brew install jq ripgrep`; the [jq download page](https://jqlang.org/download/) lists other macOS options for `jq`. If `codex` is still not found after installation, follow the installer's PATH instructions or open a new shell before continuing.

### 2. Load Qwen for Codex

```zsh
lms-load-codex
```

The loader unloads currently resident models, loads Qwen with a 200K context, maximum GPU offload, and MTP speculative decoding, assigns the stable API identifier `qwen/qwen3.6-35b-a3b`, and verifies that identifier with `lms ps --json`.

### 3. Launch local Codex

From the repository you want Codex to work on:

```zsh
codex-local
```

Inside Codex, verify the provider and model with `/status`. Use `/model` to see the registered local models and change reasoning effort.

The launcher checks `http://localhost:1234/v1/models` and refuses to start unless the selected identifier is available. It defaults to a workspace-write sandbox, user-reviewed approvals, disabled native web search and analytics, and one concurrent agent thread.

## Loading a different registered model

Override the loader model with an argument or `LMS_MAIN`:

```zsh
lms-load-codex qwen/qwen3.6-35b-a3b
LMS_MAIN=qwen/qwen3.6-35b-a3b lms-load-codex
```

Then select the same registered identifier when launching Codex:

```zsh
LMS_MAIN=another/model codex-local
codex-local --model another/model
```

Add another model's metadata to the `models` array in [models/local-models.json](models/local-models.json) before selecting it. The LM Studio API identifier and catalog slug must match. All registered entries appear in `/model`; choosing an entry that is not loaded will fail on the next model request.

## Launcher behavior

Before starting Codex, `codex-local` runs `codex-model-catalog`. The generator combines [models/local-models.json](models/local-models.json) with the concise [local model instructions](prompts/local-codex.md), validates the registry, and writes a temporary Codex catalog that is removed when the session exits.

Explicit model, sandbox, and approval flags replace launcher defaults. Other arguments are forwarded unchanged. Explicit `-c` overrides and `--search` can also replace launcher defaults, so review them before relying on the routing summary printed at startup.

Set `CODEX_LOCAL_MODEL_REGISTRY` to use an alternate registry. Relative instruction paths are resolved from that registry's directory. Registered models currently share one concise base-instructions file.

See [tools.md](tools.md) for the observed model-visible tool surface and the tools deliberately excluded from local sessions. [apply-patch.md](apply-patch.md) explains why local file edits currently use Codex's injected patch executable through `exec_command` instead of a dedicated model tool.

## Model metadata and context management

The Qwen catalog entry declares a 200K physical context and a 70 percent effective context window, giving Codex a 140K managed window without global `model_context_window` or `model_auto_compact_token_limit` overrides. Automatic compaction near that threshold still needs validation in a genuinely long session because the effective-percentage field is not publicly documented.

Qwen appears in `/model` with low, medium, and high reasoning choices. The catalog defaults to medium, and the launcher does not impose a global reasoning override that would compete with an in-session selection.

## Inspecting model catalogs and prompts

The official Codex configuration documentation describes `model_catalog_json` as a startup-loaded catalog override but does not publish the schema for individual entries. [models/model-catalog.md](models/model-catalog.md) records the schema observed in the installed Codex version, parser-required fields, defaults, and bundled OpenAI examples.

Inspect the installed Codex catalog directly:

```zsh
codex debug models --bundled | jq '.models[] | {
  slug,
  context_window,
  effective_context_window_percent,
  supported_reasoning_levels,
  shell_type,
  tool_mode,
  apply_patch_tool_type
}'
```

Generate and inspect the local catalog:

```zsh
codex-model-catalog | jq
```

From this repository's root, refresh the bundled OpenAI catalog and GPT-5.6 Sol prompt snapshots:

```zsh
codex-export-models
codex-export-prompts
```

The scripts write [models/openai-models.json](models/openai-models.json) and [prompts/gpt-5.6-sol.md](prompts/gpt-5.6-sol.md). These snapshots are version-specific; rerun both commands after upgrading Codex.

## Configuration examples

- [config/config.toml.example](config/config.toml.example) contains equivalent user-level defaults. Review and merge only the settings you want into `~/.codex/config.toml`; the repository does not install user configuration.
- [config/agents.md.example](config/agents.md.example) is a snapshot of the current global `AGENTS.md` conventions.
- [config/default.rules.example](config/default.rules.example) is a snapshot of the current global permission rules.

The `.example` suffix prevents the AGENTS and rules examples from becoming active Codex configuration when this repository is opened. Provider selection must be user-level or supplied by the launcher; Codex ignores `model_provider` in a project-local `.codex/config.toml`.

## References

- [Codex CLI](https://learn.chatgpt.com/docs/codex/cli)
- [Codex configuration reference](https://learn.chatgpt.com/docs/config-file/config-reference)
- [LM Studio `lms load`](https://lmstudio.ai/docs/cli/local-models/load)
- [LM Studio `lms ps`](https://lmstudio.ai/docs/cli/local-models/ps)
