# Hybrid local Codex implementation plan

This is the durable implementation record for the hybrid setup. Phases 1 and 2 are implemented; later phases are proposals. Use the [root README](../README.md) and [Codex README](../codex/README.md) as the operational source of truth for features that exist today.

## Goal

Add a Codex setup alongside the existing Claude Code setup. Daily coding,
tool use, and ordinary subagents should use `qwen/qwen3.6-35b-a3b` through LM
Studio. OpenAI use should remain explicit so it is clear which provider handles
a workflow:

- `gpt-5.6-sol` for selected large-refactor planning sessions.
- The Responses API web-search tool behind a narrow MCP server.

This is not intended to create a hard isolation boundary between local and
OpenAI-backed Codex sessions. The narrow web-search path should still receive
only its declared query or URL rather than the Codex transcript or repository
contents. The existing Claude behavior and commands must remain available.

## Target layout

The tree below includes planned Phase 3 and Phase 4 files, marked accordingly.

```text
llms/
├── AGENTS.md
├── README.md
├── .gitignore
├── bin/                         # User-facing commands for both setups
│   ├── claude-local
│   ├── claude-local-sessions
│   ├── lms-load
│   ├── lms-session-stats
│   ├── codex-local
│   ├── codex-model-catalog
│   ├── codex-export-models
│   ├── codex-export-prompts
│   ├── codex-cloud-plan             # Planned
│   └── lms-load-codex
├── claude/
│   ├── README.md
│   └── todo.md
├── codex/
│   ├── README.md
│   ├── apply-patch.md
│   ├── tools.md
│   ├── models/
│   │   ├── model-catalog.md
│   │   ├── openai-models.json
│   │   └── local-models.json
│   ├── config/
│   │   ├── config.toml.example
│   │   ├── agents.md.example
│   │   ├── default.rules.example
│   │   └── cloud-plan.config.toml.example # Planned
│   ├── prompts/
│   │   ├── local-codex.md
│   │   └── gpt-5.6-sol.md
│   └── web-mcp/                     # Planned
│       ├── package.json
│       ├── package-lock.json
│       ├── tsconfig.json
│       ├── src/
│       │   ├── index.ts
│       │   ├── search-web.ts
│       │   ├── fetch-url.ts
│       │   ├── safe-fetch.ts
│       │   ├── extract-content.ts
│       │   └── privacy.ts
│       └── test/
│           ├── search-web.test.ts
│           ├── fetch-url.test.ts
│           ├── safe-fetch.test.ts
│           └── privacy.test.ts
├── docs/
│   ├── codex-hybrid-plan.md
│   ├── models.md
│   └── todo.md
└── ollama/                          # Archived Ollama-first implementation
    ├── README.md
    ├── modelfiles/
    └── ollama-session-stats
```

Keep the real implementations in the top-level `bin/` so PATH entries and
optional `~/.local/bin` links have one predictable target. New Codex links must
be created explicitly; the repository does not install or overwrite them.

## Phase 1: Reorganize without changing Claude behavior

Status: implemented.

Move the current Claude-specific material under `claude/`:

- Move the existing README to `claude/README.md`.
- Move the model analysis to `docs/models.md` so both client guides can reference it.
- Move Claude-specific backlog items to `claude/todo.md`.
- Keep the scripts in the top-level `bin/` and the archived Ollama material in top-level `ollama/` without changing behavior.
- Keep `.claude/settings.local.json` at the repository root so Claude sessions
  started there retain their current project-local behavior. Document why it
  remains outside the otherwise split Claude subtree.

Rewrite the root README as a shared setup guide and index to the Claude and Codex subtrees. Keep cross-cutting model research in `docs/todo.md`.

Acceptance criteria:

- Existing Claude `~/.local/bin` symlinks remain valid.
- `claude-local`, `lms-load`, and the monitoring commands behave identically.
- No Claude model IDs, flags, environment variables, or runtime behavior change
  as part of the move.

## Phase 2: Add the minimal local Codex path

Status: implemented.

Create `bin/lms-load-codex` that:

1. Finds the `lms` executable.
2. Unloads currently loaded models.
3. Loads only `qwen/qwen3.6-35b-a3b`.
4. Uses a 200K context and the current MLX/MTP options.
5. Verifies the loaded identifier through `lms ps`.

Create `bin/codex-local` that:

1. Checks `http://localhost:1234/v1/models`.
2. Verifies that the requested model is available.
3. Defaults `LMS_MAIN` to `qwen/qwen3.6-35b-a3b`.
4. Runs Codex with `--oss --local-provider lmstudio`.
5. Supplies the generated model catalog plus explicit sandbox, approval, and web-search settings.
6. Preserves command-line overrides where practical.

The following was the initial launcher design. The implemented custom catalog supersedes its global context, compaction, and reasoning settings so each registered local model can declare its own metadata:

```toml
model_provider = "lmstudio"
model = "qwen/qwen3.6-35b-a3b"
model_context_window = 200000
model_auto_compact_token_limit = 140000
model_reasoning_effort = "medium"

sandbox_mode = "workspace-write"
approval_policy = "on-request"
approvals_reviewer = "user"
web_search = "disabled"

[analytics]
enabled = false

[agents]
enabled = true
default_subagent_model = "qwen/qwen3.6-35b-a3b"
default_subagent_reasoning_effort = "medium"
max_concurrent_threads_per_session = 1
```

Document a user-level `~/.codex/config.toml` alternative in
`codex/config/config.toml.example`, but do not install or merge it
automatically. Machine-local provider settings cannot be supplied by a
project-local `.codex/config.toml`.

Do not force `model_supports_reasoning_summaries` initially. Test LM Studio's
Responses behavior first and disable reasoning summaries only if required.

### Local model catalog follow-up

Status: implemented for the initial Qwen entry, with a documented freeform-tool compatibility gap.

Add a custom local model catalog before treating Phase 2 as the long-term
launcher design. The catalog should:

- Give every supported local model its own identifier, display name, context
  window, effective-context percentage, reasoning levels, modalities, concise
  base instructions, and direct tool-protocol metadata.
- Use the unified shell protocol rather than code-mode-only orchestration. The registry currently declares the freeform `apply_patch` protocol, but the dedicated tool is not model-visible in the tested LM Studio/Qwen session. Local instructions therefore invoke Codex's injected patch executable through `exec_command`. An A/B check should confirm that the installed CLI still injects the helper without the catalog field, after which the unusable declaration can be removed; see [codex/apply-patch.md](../codex/apply-patch.md).
- Allow multiple already-loaded LM Studio models to appear in `/model`.
- Keep provider routing explicit without trying to enforce a security boundary
  between local and OpenAI sessions.
- Implement and verify the curated tool surface recorded in
  [codex/tools.md](../codex/tools.md).

The catalog now supplies Qwen's 200K context window and a 70 percent effective
window, and the launcher no longer supplies global context, compaction, or
reasoning overrides. There is no documented per-model absolute
`model_auto_compact_token_limit` field in the catalog. Verify that Codex
automatically compacts near the intended 140K effective window. If it does not,
retain a session-wide launcher override or use separate profiles for models
with materially different limits.

A synthetic request capture contained nine tool entries, but the live Qwen session reported seven: `exec_command`, `write_stdin`, `request_user_input`, `view_image`, `get_goal`, `create_goal`, and `update_goal`. Dedicated `apply_patch` and the multi-agent namespace were absent. Patch create, update, and delete behavior is verified through the injected shell helper; the multi-agent discrepancy remains open.

## Phase 3: Add the explicit cloud planner

Status: not started.

Create `bin/codex-cloud-plan` and
`codex/config/cloud-plan.config.toml.example`.

The cloud planner should always:

- Use the OpenAI provider and `gpt-5.6-sol`.
- Use high reasoning effort.
- Start a new thread rather than resume or fork a local thread.
- Use a read-only sandbox with `approval_policy = "never"`.
- Disable the hybrid web MCP and native web search unless explicitly enabled.
- Disable subagents by default.
- Clearly print that the request will be sent to OpenAI.

Support two documented workflows:

1. Run read-only against a repository, accepting that inspected repository
   content will be sent to OpenAI.
2. Run against a temporary planning packet containing only selected files and
   a problem statement.

The separate-thread default is an operational guardrail against accidentally carrying a local transcript into a cloud planning request, not a hard privacy boundary. An operator may intentionally run the planner against a repository or selected files after accepting that those inputs will be sent to OpenAI.

The output should be Markdown suitable for saving as a plan artifact that a
local Codex session can implement.

## Phase 4: Implement the web MCP

Status: not started.

Use TypeScript with the official MCP SDK and OpenAI SDK. Expose only two tools:

```text
search_web
fetch_url
```

### `search_web`

- Accept a query, optional allowed domains, recency, and result count.
- Cap query length.
- Reject likely credentials and oversized pasted code.
- Call the OpenAI Responses API with hosted `web_search`.
- Set `store: false`.
- Send no Codex transcript or repository context.
- Return a concise summary and structured source list.
- Avoid logging raw queries by default.
- Mock the OpenAI client in tests and assert that no undeclared context reaches
  the API request.

The boundary cannot be absolute if the local model controls the query string.
The MCP server must therefore omit any generic `context` argument, apply
content and length checks, and make tool arguments visible for approval during
the initial rollout.

### `fetch_url`

- Fetch directly from the MCP process so the local Qwen model performs the
  summarization.
- Initially support HTML, Markdown, plain text, and bounded JSON.
- Reject credentials embedded in URLs.
- Reject loopback, link-local, multicast, and private destinations.
- Revalidate every redirect.
- Pin validated destinations where feasible to reduce DNS-rebinding risk.
- Limit redirects, response bytes, decompressed size, duration, and returned
  text.
- Strip scripts, styles, forms, navigation, and hidden content.
- Return extracted text with the final URL and truncation metadata.
- Label page content as untrusted external data.

Defer PDF support until it has a defined parser and explicit resource limits.

Example MCP registration:

```toml
[mcp_servers.hybrid_web]
command = "node"
args = ["/absolute/path/to/llms/codex/web-mcp/dist/index.js"]
env_vars = ["OPENAI_API_KEY"]
required = true
enabled_tools = ["search_web", "fetch_url"]
default_tools_approval_mode = "prompt"
tool_timeout_sec = 60

[mcp_servers.hybrid_web.tools.search_web]
output_token_limit = 12000

[mcp_servers.hybrid_web.tools.fetch_url]
output_token_limit = 24000
```

Keep every web call interactive initially. Relax approval behavior only after
reviewing real tool arguments and egress behavior.

## Phase 5: Add deterministic command rules

Status: not started. The current example snapshot is `codex/config/default.rules.example`; it is not installed automatically.

Create a documented rules example rather than installing it automatically.
It should:

- Forbid mutating Git subcommands while allowing read-only inspection.
- Forbid direct database clients.
- Prompt for package installation and broad network utilities.
- Permit narrow project-specific lint and test prefixes where appropriate.

Test every example using `codex execpolicy check`. Keep
`approvals_reviewer = "user"` for the initial release. Record auto-review as a
future experiment; do not enable it until its provider and transmitted context
have been observed. Auto-review is unnecessary for routine commands that stay
inside the OS-enforced sandbox.

## Phase 6: Documentation and operational hardening

Status: in progress. Shared setup and the implemented local Codex path are documented; the cloud planner and web MCP sections remain pending until those features exist.

The root README should cover installation, shared LM Studio setup, model download, server startup, and command discovery. `codex/README.md` should cover:

- Loading the one-model Codex set.
- Local daily use.
- Cloud planning and privacy boundaries.
- Web MCP registration.
- API-key handling.
- Sandbox and approval modes.
- Subagent memory implications.
- Troubleshooting Responses tool calls.
- Inspecting LM Studio logs and Codex JSON output.

Add measured Codex behavior to `docs/models.md`, or to a dedicated benchmark document if it becomes large, rather than repeating general model descriptions:

- First-turn prefill.
- Decode rate.
- Tool-call success rate.
- Patch correctness.
- Compaction behavior.
- Context-cache restoration.
- One versus two subagent KV-cache cost.
- Reasoning-summary compatibility.

Extend `.gitignore` with:

```gitignore
node_modules/
dist/
coverage/
.env
.env.*
!.env.example
```

Never store an OpenAI API key or copied Codex authentication state in the
repository.

## Verification required during implementation

The root `AGENTS.md` is the source of truth for current checks. At the time of this plan, they are:

```sh
zsh -n bin/claude-local bin/lms-load
bash -n bin/codex-local bin/codex-model-catalog bin/codex-export-models bin/codex-export-prompts bin/lms-load-codex
jq empty codex/models/local-models.json
bin/codex-model-catalog | jq empty
PYTHONPYCACHEPREFIX=/private/tmp/llms-pycache python3 -m py_compile bin/claude-local-sessions bin/lms-session-stats ollama/ollama-session-stats
for script in bin/* ollama/ollama-session-stats; do test -x "$script" || exit 1; done
```

When Phase 4 exists, add its package install, lint, type-check, unit-test, and build commands to `AGENTS.md`. Its unit tests must mock LM Studio and OpenAI integrations.

Manual, non-database smoke tests:

```sh
claude-local --version
codex --version
codex-model-catalog | jq empty
lms server status
```

LM Studio and OpenAI integration tests must be opt-in because they require
running services, network access, or billable API calls. Unit tests must mock
both. Do not run anything that may connect to a database without explicit user
approval.

## Recommended implementation order

1. Done: reorganize the Claude files while keeping all commands in the top-level `bin/`.
2. Done: add the one-model Codex loader, local launcher, model registry, and generated catalog.
3. Add the cloud-planner launcher and configuration example.
4. Finish smoke-testing Qwen's Responses tool calling, patching, compaction, and subagents.
5. Implement and test `fetch_url`.
6. Implement and test OpenAI-backed `search_web`.
7. Add and test deterministic Git, database, and network rules.
8. Finish the documentation and measured-model notes.
9. Investigate auto-review separately after the base system is stable.
