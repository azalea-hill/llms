# Hybrid local Codex implementation plan

## Goal

Add a Codex setup alongside the existing Claude Code setup. Daily coding,
tool use, and ordinary subagents should use `qwen/qwen3.6-35b-a3b` through LM
Studio. OpenAI should be used only through explicit, isolated paths:

- `gpt-5.6-sol` for selected large-refactor planning sessions.
- The Responses API web-search tool behind a narrow MCP server.

The OpenAI web-search path must not receive the Codex session transcript or
repository contents. The existing Claude behavior and commands must remain
available.

## Target layout

```text
llms/
├── AGENTS.md
├── README.md
├── TODO.md
├── .gitignore
├── .claude/                     # Retained for sessions started at repo root
│   └── settings.local.json
├── bin/                         # User-facing commands for both setups
│   ├── claude-local
│   ├── claude-local-sessions
│   ├── lms-load
│   ├── lms-session-stats
│   ├── codex-local
│   ├── codex-cloud-plan
│   └── lms-load-codex
├── claude/
│   ├── README.md
│   ├── MODELS.md
│   ├── TODO.md
│   └── ollama/
│       ├── modelfiles/
│       └── ollama-session-stats
└── codex/
    ├── README.md
    ├── MODELS.md
    ├── TODO.md
    ├── config/
    │   ├── config.toml.example
    │   ├── cloud-plan.config.toml.example
    │   └── rules/
    │       └── default.rules
    └── web-mcp/
        ├── package.json
        ├── package-lock.json
        ├── tsconfig.json
        ├── src/
        │   ├── index.ts
        │   ├── search-web.ts
        │   ├── fetch-url.ts
        │   ├── safe-fetch.ts
        │   ├── extract-content.ts
        │   └── privacy.ts
        └── test/
            ├── search-web.test.ts
            ├── fetch-url.test.ts
            ├── safe-fetch.test.ts
            └── privacy.test.ts
```

The existing `~/.local/bin` links point into the top-level `bin/`. Keep the real
implementations there so those links remain stable and all user-facing commands
have one predictable home.

## Phase 1: Reorganize without changing Claude behavior

Move the current Claude-specific material under `claude/`:

- Move the existing README to `claude/README.md`.
- Move the model analysis to `claude/MODELS.md`.
- Move Claude-specific backlog items to `claude/TODO.md`.
- Keep the scripts in the top-level `bin/` and move the archived Ollama material
  under `claude/` without changing behavior.
- Keep `.claude/settings.local.json` at the repository root so Claude sessions
  started there retain their current project-local behavior. Document why it
  remains outside the otherwise split Claude subtree.

Rewrite the root README as a short guide to the Claude and Codex subtrees.
Keep cross-cutting model research in the root TODO.

Acceptance criteria:

- Existing `~/.local/bin` symlinks remain valid.
- `claude-local`, `lms-load`, and the monitoring commands behave identically.
- No Claude model IDs, flags, environment variables, or runtime behavior change
  as part of the move.

## Phase 2: Add the minimal local Codex path

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
5. Supplies explicit context, compaction, sandbox, approval, and web-search
   settings.
6. Preserves command-line overrides where practical.

The initial defaults should be:

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

## Phase 3: Add the isolated cloud planner

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

Privacy invariant:

> Never switch an existing local thread to the cloud profile. Never resume or
> fork a local thread using the cloud planner.

The output should be Markdown suitable for saving as a plan artifact that a
local Codex session can implement.

## Phase 4: Implement the web MCP

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

`codex/README.md` should cover:

- Installation and LM Studio startup.
- Loading the one-model Codex set.
- Local daily use.
- Cloud planning and privacy boundaries.
- Web MCP registration.
- API-key handling.
- Sandbox and approval modes.
- Subagent memory implications.
- Troubleshooting Responses tool calls.
- Inspecting LM Studio logs and Codex JSON output.

`codex/MODELS.md` should record measured behavior rather than repeat general
model descriptions:

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

Record the final, exact commands in the closest applicable `AGENTS.md`. The
expected checks are:

```sh
zsh -n \
  bin/claude-local \
  bin/lms-load \
  bin/codex-local \
  bin/codex-cloud-plan \
  bin/lms-load-codex

python3 -m py_compile \
  bin/claude-local-sessions \
  bin/lms-session-stats \
  claude/ollama/ollama-session-stats

npm --prefix codex/web-mcp ci
npm --prefix codex/web-mcp run lint
npm --prefix codex/web-mcp run typecheck
npm --prefix codex/web-mcp test
npm --prefix codex/web-mcp run build
```

Manual, non-database smoke tests:

```sh
claude-local --version
codex --version
codex execpolicy check --pretty --rules codex/config/rules/default.rules -- git status
codex execpolicy check --pretty --rules codex/config/rules/default.rules -- git commit -m test
```

LM Studio and OpenAI integration tests must be opt-in because they require
running services, network access, or billable API calls. Unit tests must mock
both. Do not run anything that may connect to a database without explicit user
approval.

## Recommended implementation order

1. Reorganize the Claude files while keeping all commands in the top-level `bin/`.
2. Add the one-model Codex loader and local launcher.
3. Add local and cloud-planner configuration examples.
4. Smoke-test Qwen's Responses tool calling, patching, compaction, and
   subagents.
5. Implement and test `fetch_url`.
6. Implement and test OpenAI-backed `search_web`.
7. Add and test deterministic Git, database, and network rules.
8. Finish the documentation and measured-model notes.
9. Investigate auto-review separately after the base system is stable.
