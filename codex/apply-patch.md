# `apply_patch` with local Codex

## Current result

With Codex CLI 0.154.0, `qwen/qwen3.6-35b-a3b`, and LM Studio, the dedicated freeform `apply_patch` tool is declared in the custom model catalog but is not available in the live model-visible tool set. Codex's injected `apply_patch` executable does work through `exec_command`, including native file deletion.

The practical local editing path is therefore:

```text
Qwen → exec_command → Codex-injected apply_patch executable → Codex patch parser → workspace
```

The intended but currently unavailable path is:

```text
Qwen → dedicated freeform apply_patch tool call → Codex patch parser → workspace
```

This is a provider/model compatibility finding, not evidence that Codex's patch parser lacks create, update, or delete support.

## Configuration evidence

The generated Qwen catalog and the bundled `gpt-5.6-sol` catalog both declare:

```json
{
  "apply_patch_tool_type": "freeform",
  "shell_type": "unified_exec",
  "experimental_supported_tools": []
}
```

The local base instructions also named `apply_patch`. An earlier synthetic request capture contained a request-level `apply_patch` definition, but a later live Qwen session reported only:

```text
exec_command
write_stdin
request_user_input
view_image
get_goal
create_goal
update_goal
```

It explicitly answered `apply_patch unavailable` when instructed to make exactly one direct `apply_patch` call. The missing multi-agent namespace is a separate discrepancy recorded in [tools.md](tools.md).

OpenAI documents `apply_patch` as a freeform call rather than an ordinary JSON function call and notes that its models are specifically trained for the tool. The evidence below now makes the provider's handling of non-function tool types more likely than a Qwen-only problem, although we have not captured LM Studio's internal rendered prompt and cannot call that proven.

## Related reports and likely cause

This is not an isolated result:

- [OpenAI Codex issue #17899](https://github.com/openai/codex/issues/17899) reports `apply_patch` failures with Codex CLI 0.120.0, `gpt-oss-20b`, LM Studio, and macOS. It remains open as of September 15, 2026.
- [OpenAI Codex issue #15899](https://github.com/openai/codex/issues/15899) reports a local `gpt-oss-20b` model trying to call an unavailable `apply_patch`; it was closed as a duplicate of a broader tool-call issue.
- [Unsloth issue #9114](https://github.com/unslothai/unsloth/issues/9114) reproduces the sharper form of our result across multiple local models: the prompt mentions `apply_patch`, but the actual tool list omits it while shell tools remain available.
- The investigation in [Unsloth PR #9121](https://github.com/unslothai/unsloth/pull/9121) captured Codex's request and found that `apply_patch_tool_type: "freeform"` adds a `type: "custom"` tool with a Lark grammar, but that integration's Responses-to-chat translator forwarded only `type: "function"` tools. The merged fix added an explicit bridge for the custom call and its streamed/replayed outputs.
- [LM Studio's Responses documentation](https://lmstudio.ai/docs/developer/openai-compat/responses) documents ordinary Responses requests and Remote MCP tools but does not claim support for OpenAI custom/freeform or native apply-patch calls. [LM Studio issue #1810](https://github.com/lmstudio-ai/lmstudio-bug-tracker/issues/1810) separately confirms that at least some non-function Codex tool types have failed validation in its Responses compatibility layer.

The Unsloth translator is not LM Studio, so its exact code path does not prove LM Studio's root cause. Together with our request-level capture and live seven-tool result, however, it strongly suggests the freeform tool is being lost or made unusable at the Responses compatibility or prompt-rendering boundary before Qwen can call it. I found no documented LM Studio setting that enables Codex's custom/freeform apply-patch protocol.

## Smoke-test history

The tests were performed around Codex session `01a0a655-a48e-7032-800e-8b6b98acbaa2` and a disposable `/tmp/codex-apply-patch-test` workspace.

| Test | Result | What it established |
| --- | --- | --- |
| Create, modify, and revert with standard unified diffs | Succeeded | Qwen can generate usable diffs, but this alone does not prove use of the dedicated tool. |
| Delete with a unified diff targeting `/dev/null` | Emptied the file without unlinking it | The particular patch execution path treated the diff as content removal. |
| Pass `*** Delete File` to Unix `patch` | Rejected with exit code 2 | Expected: Codex's native directives are not Unix unified-diff syntax. |
| Invoke the Codex-injected `apply_patch` executable through `exec_command` | Deleted the file | Codex's native parser and delete directive work end to end through the shell wrapper. |
| Request the dedicated `apply_patch` tool from the correct workspace | Model reported it unavailable | The direct freeform tool was not in the live tool set recognized by Qwen. |

The working-directory correction matters. Native patch paths are relative to the tool call's `workdir`; absolute paths and Git-style `a/` or `b/` prefixes should not be used with `*** Add File`, `*** Update File`, or `*** Delete File` directives.

## Supported local workaround

Use `exec_command` with its `workdir` set to the target workspace and invoke the injected helper, not the Unix `patch` program:

```sh
apply_patch <<'PATCH'
*** Begin Patch
*** Update File: relative/path.txt
@@
-old text
+new text
*** End Patch
PATCH
```

Creation and deletion use the corresponding native directives:

```text
*** Add File: relative/path.txt
*** Delete File: relative/path.txt
```

The updated [local base instructions](prompts/local-codex.md) tell Qwen to use this route for deliberate file changes, use paths relative to `workdir`, and avoid both Unix `patch` and shell-writing shortcuts.

This path is not an invented third-party executable. Current [Codex source](https://github.com/openai/codex/blob/main/codex-rs/arg0/src/lib.rs) creates a per-session `apply_patch` symlink to the Codex executable on Unix, prepends its directory to `PATH`, and dispatches it to Codex's internal patch implementation. Codex also ships an [apply-patch shell-command prompt template](https://github.com/openai/codex/blob/main/codex-rs/prompts/templates/apply_patch_tool_instructions.md). That makes this a supported Codex fallback path, even though it is not the dedicated model tool.

## Workaround tradeoffs

| Area | Effect |
| --- | --- |
| Editing capability | Create, update, move, and delete still use Codex's native patch parser. The verified operations are not downgraded to Unix `patch` semantics. |
| Local-model compatibility | Qwen only has to make the ordinary `exec_command` function call it already handles. This is likely more reliable than asking it to emit a freeform custom-tool call that the provider does not expose. |
| Prompt and context | The model needs a short shell-wrapper instruction, but the unusable dedicated-tool grammar does not need to be model-visible. Removing that declaration after the A/B check may slightly reduce request size and eliminate a contradictory tool instruction. |
| Shell syntax | The patch is carried inside a heredoc, so a line equal to the chosen delimiter would terminate it early. The command should begin directly with `apply_patch`, use a single-quoted delimiter, and avoid `cd &&`, alternate spellings, or nested shell wrappers. Older Codex versions have had interception failures for unusual command shapes. |
| Permissions | The call travels through `exec_command`, so it remains subject to Codex's shell sandbox and approval policy. That preserves the important safety boundary, but it can produce shell-style approval or error reporting instead of patch-specific reporting. |
| Tool results and replay | The model receives a generic command result rather than a dedicated typed patch-call result. This can make telemetry, streamed event handling, retries, and transcript diagnosis less precise even though ordinary iterative editing works. |
| Portability | The helper is per-session and supplied by the Codex executable. Scripts outside a running Codex environment should not assume `apply_patch` exists, and the behavior should be rechecked after Codex upgrades. |
| Performance | There is a small extra shell/process-dispatch cost and some wrapper tokens. Compared with local-model inference and repository inspection, it should be negligible. |

The most important operational tradeoff is syntax reliability, not loss of patch features or sandboxing. Keep the exact, simple heredoc form in the prompt and do not teach the model several equivalent spellings.

## Catalog flag decision

Current Codex source indicates that the PATH helper is created during CLI startup independently of model-catalog tool registration. Therefore `apply_patch_tool_type: "freeform"` is probably not needed for the working shell path; its job is to request the dedicated freeform tool.

Keep the field only until one short A/B test confirms that the installed Codex 0.154.0 still injects the helper when the field is omitted. If `command -v apply_patch` and create/update/delete tests continue to pass, remove the field from the local entry so the request does not advertise a tool the live provider/model path cannot use. Do not change it to `"function"`: recent Codex catalog parsers accept `"freeform"` but reject that value, and a standard JSON function would require an actual translation layer rather than a catalog spelling change.

## Heavier alternative

A compatibility proxy can preserve the dedicated semantics by translating Codex's custom/freeform `apply_patch` definition into a model-visible function, then translating the model's function call and result back into the Responses custom-tool event stream. The merged Unsloth bridge demonstrates that this is feasible, including streaming and replay handling.

That is not a drop-in LM Studio setting. It adds another stateful protocol component that must correctly handle tool schemas, call IDs, streaming events, retries, and permission boundaries. For this proof of concept, the shell-helper route is the better default unless generic custom-tool support becomes important beyond file editing.

## Next diagnostics

1. Compare otherwise identical local sessions with and without `apply_patch_tool_type`, check `command -v apply_patch`, and repeat create/update/delete through `exec_command`.
2. If the helper is unchanged as expected, omit `apply_patch_tool_type` from the local catalog and confirm the contradictory direct tool disappears from the request-level capture.
3. Capture only tool type/name fields at LM Studio's request boundary to establish whether it receives `{ "type": "custom", "name": "apply_patch" }`.
4. Determine whether LM Studio includes that custom tool in the prompt or tool grammar rendered for Qwen.
5. Re-test native custom-tool support after Codex or LM Studio upgrades; consider a protocol bridge only if more custom tools require it.

Codex session transcripts normally live under `~/.codex/sessions` and `~/.codex/archived_sessions`. Treat them as sensitive: they can include prompts, repository content, paths, and tool arguments. Extract only the fields needed for a diagnosis rather than sharing an entire transcript.

For a known session ID, this `find`-based check avoids requiring ripgrep and emits only tool names from matching response items:

```sh
session_id=01a0a655-a48e-7032-800e-8b6b98acbaa2
session_file=$(find "$HOME/.codex/sessions" "$HOME/.codex/archived_sessions" -type f -name "*$session_id*.jsonl" -print -quit 2>/dev/null)
if [ -n "$session_file" ]; then
  jq -r 'select(.type == "response_item") | .payload | select(.type == "function_call" or .type == "custom_tool_call") | [.type, .name] | @tsv' "$session_file"
else
  printf '%s\n' "session not found"
fi
```

This deliberately does not print arguments, prompts, command output, or other transcript payloads. Transcript formats can change between Codex versions; if it emits nothing, inspect the installed format locally and continue to avoid printing unstructured payloads.

## References

- [OpenAI model guidance: `apply_patch` and freeform custom tools](https://developers.openai.com/api/docs/guides/latest-model?model=gpt-5.2)
- [OpenAI Apply Patch guide](https://developers.openai.com/api/docs/guides/tools-apply-patch)
- [OpenAI Responses API reference](https://developers.openai.com/api/reference/cli/resources/beta/subresources/responses)
- [Codex troubleshooting and transcript locations](https://learn.chatgpt.com/docs/reference/troubleshooting)
- [LM Studio Codex integration](https://lmstudio.ai/docs/integrations/codex)
- [LM Studio Responses compatibility endpoint](https://lmstudio.ai/docs/developer/openai-compat/responses)
- [OpenAI Codex local-provider issue #17899](https://github.com/openai/codex/issues/17899)
- [Unsloth local-provider issue #9114 and merged bridge #9121](https://github.com/unslothai/unsloth/pull/9121)
