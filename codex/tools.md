# Local Codex tool surface

This document records the model-visible tools observed with local models. The goal is to give them the coding tools they can use reliably without spending context on unrelated Codex Apps, connectors, or experimental protocols. This is a context-efficiency and reliability choice, not a hard isolation boundary from OpenAI services.

## Status

The curated surface is configured by `bin/codex-local`, the local model registry, and the generated custom catalog. Codex CLI 0.154.0 produced different results at two observation points:

- A synthetic request capture with the custom catalog contained nine entries and approximately 17,197 characters of serialized definitions. Those entries included the freeform `apply_patch` tool and the `multi_agent_v1` namespace.
- A live `qwen/qwen3.6-35b-a3b` session through LM Studio reported only the seven tools listed below. It explicitly reported that `apply_patch` was unavailable, and it did not list the multi-agent namespace.

This discrepancy suggests that the request-level catalog is not identical to the tool grammar ultimately available to Qwen. Reports from other local-provider integrations show the same symptom when a Responses compatibility layer drops non-function/custom tools. That makes the provider or prompt-rendering boundary the leading explanation, but it does not prove LM Studio's internal path. See [apply-patch.md](apply-patch.md) for the external evidence, tradeoffs, current workaround, and next diagnostics.

The exact surface also depends on the user's enabled MCP configuration. The launcher disables built-in Apps and Plugins but does not erase explicitly configured third-party MCP servers. Such a server can add model-visible tools and must be reviewed separately.

## Observed model-visible tools

The live local session reported:

- `exec_command`: run shell commands, inspect files, and invoke Codex's injected `apply_patch` executable. It covers the practical equivalents of Claude Code's Bash, Read, Write, Glob, Grep, and Edit tools.
- `write_stdin`: continue or interact with a command started by `exec_command`.
- `request_user_input`: ask the user for a structured choice when available in the active Codex mode.
- `view_image`: inspect an image already present in the workspace.
- `get_goal`, `create_goal`, and `update_goal`: the closest counterpart to Claude Code's TodoWrite workflow.

Plan-mode entry and exit are Codex UI/session controls, not model-callable tools, so they do not need tool schemas in every inference request.

## Declared but unavailable in the live session

- Dedicated `apply_patch`: `codex/models/local-models.json` declares `apply_patch_tool_type: "freeform"`, matching the bundled OpenAI entries. The live Qwen session did not receive or recognize it as a callable tool. File creation, modification, and deletion work through `exec_command` calling Codex's injected `apply_patch` executable.
- Multi-agent tools: the catalog declares `multi_agent_version: "v1"` and the launcher enables agents with one concurrent thread, but the live session did not list a multi-agent namespace. Diagnose this separately from patch handling.
- `search_web` and `fetch_url`: these remain planned tools for the narrow hybrid web MCP and are not implemented.

Current Codex source creates the shell helper independently of the model catalog, so `apply_patch_tool_type: "freeform"` is unlikely to be necessary for the fallback. Keep it only until the installed 0.154.0 build passes the short A/B test described in [apply-patch.md](apply-patch.md); then remove it if the helper remains available.

## Deliberately excluded

The local profile should not advertise these tools or namespaces by default:

- Native Codex web search. The planned hybrid web MCP provides the intentionally narrow `search_web` and `fetch_url` interface instead.
- Codex Apps namespaces observed in the fallback request: `codex_document_control`, `hotline`, `plugin_management`, `sites`, and `workspace_agents`.
- Other Apps, connectors, and plugins inherited from the user's general Codex configuration unless explicitly added to the local profile.
- Computer-use and browser-control tools.
- Image-generation tools.
- Code-mode-only or JavaScript orchestration protocols. Local models use direct tools and the unified shell protocol.
- Arbitrary MCP servers and their resource-browsing tools. Enable only servers deliberately selected for this workflow.

Exclusion from model-visible tool calling does not necessarily remove the underlying application feature from Codex or the machine. It means the local model should not receive that tool's schema or be able to invoke it in this profile.

## Verification after upgrades

Repeat the following after upgrading Codex, LM Studio, or Qwen:

- Capture the request-level tool names and serialized definition sizes.
- Ask the live model to enumerate its callable tool names.
- Test the injected `apply_patch` executable for create, update, and delete operations.
- Test whether the dedicated freeform `apply_patch` tool becomes visible.
- Test whether the multi-agent namespace becomes visible.
- Confirm that excluded Apps and native web-search tools remain absent.

Treat all measurements as version-specific. Review session transcripts before sharing them because they may contain prompts, repository content, paths, and tool arguments.
