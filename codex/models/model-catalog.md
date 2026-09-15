# Codex model catalog schema

This is a version-specific reference for the JSON accepted by
`model_catalog_json`. It was derived from Codex CLI 0.154.0 on 2026-09-15 by
running the requested command:

```zsh
codex debug models --bundled | jq '.models[]'
```

The bundled output was 523,401 bytes and contained 11 model entries. The
official Codex configuration reference documents `model_catalog_json` as a
startup-loaded catalog override, but does not publish the schema of individual
entries. Treat this document as observed behavior, not a stable OpenAI API.

## Reproduce the source data

Print the entire catalog:

```zsh
codex debug models --bundled | jq
```

Print one complete entry, including its full embedded instructions:

```zsh
codex debug models --bundled |
  jq '.models[] | select(.slug == "gpt-5.6-sol")'
```

Print the fields most relevant to a local model catalog:

```zsh
codex debug models --bundled | jq '.models[] | {
  slug,
  display_name,
  context_window,
  max_context_window,
  effective_context_window_percent,
  default_reasoning_level,
  supported_reasoning_levels,
  shell_type,
  tool_mode,
  apply_patch_tool_type,
  supports_search_tool,
  web_search_tool_type,
  input_modalities
}'
```

`model_catalog_json` replaces the catalog loaded for that Codex process. A
custom catalog therefore needs entries for every model that should appear in
that process's model picker.

## Observed JSON Schema

This descriptive JSON Schema captures the types and required fields accepted by
the 0.154.0 parser. `additionalProperties` is intentionally allowed because the
format is internal and later Codex releases may add fields. Enumerated values
are recorded separately below rather than treated as an exhaustive contract.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Observed Codex 0.154.0 model catalog",
  "type": "object",
  "required": ["models"],
  "properties": {
    "models": {
      "type": "array",
      "items": { "$ref": "#/$defs/model" }
    }
  },
  "additionalProperties": true,
  "$defs": {
    "model": {
      "type": "object",
      "required": [
        "slug",
        "display_name",
        "supported_reasoning_levels",
        "shell_type",
        "visibility",
        "supported_in_api",
        "priority",
        "support_verbosity",
        "truncation_policy",
        "experimental_supported_tools"
      ],
      "properties": {
        "slug": { "type": "string" },
        "display_name": { "type": "string" },
        "description": { "type": ["string", "null"] },
        "default_reasoning_level": { "type": ["string", "null"] },
        "supported_reasoning_levels": {
          "type": "array",
          "items": { "$ref": "#/$defs/reasoning_level" }
        },
        "shell_type": { "type": "string" },
        "visibility": { "type": "string" },
        "supported_in_api": { "type": "boolean" },
        "priority": { "type": "integer" },
        "additional_speed_tiers": {
          "type": "array",
          "items": { "type": "string" }
        },
        "service_tiers": {
          "type": "array",
          "items": { "$ref": "#/$defs/service_tier" }
        },
        "availability_nux": { "type": ["object", "null"] },
        "upgrade": {
          "anyOf": [{ "$ref": "#/$defs/upgrade" }, { "type": "null" }]
        },
        "base_instructions": { "type": "string" },
        "model_messages": { "$ref": "#/$defs/model_messages" },
        "include_skills_usage_instructions": { "type": "boolean" },
        "include_plugin_usage_instructions": { "type": "boolean" },
        "include_apps_usage_instructions": { "type": "boolean" },
        "default_reasoning_summary": { "type": "string" },
        "support_verbosity": { "type": "boolean" },
        "default_verbosity": { "type": ["string", "null"] },
        "apply_patch_tool_type": { "type": ["string", "null"] },
        "web_search_tool_type": { "type": "string" },
        "truncation_policy": { "$ref": "#/$defs/truncation_policy" },
        "supports_image_detail_original": { "type": "boolean" },
        "context_window": { "type": "integer" },
        "max_context_window": { "type": "integer" },
        "comp_hash": { "type": "string" },
        "effective_context_window_percent": { "type": "integer" },
        "experimental_supported_tools": {
          "type": "array",
          "items": { "type": "string" }
        },
        "input_modalities": {
          "type": "array",
          "items": { "type": "string" }
        },
        "supports_search_tool": { "type": "boolean" },
        "supports_experimental_context": { "type": "boolean" },
        "use_responses_lite": { "type": "boolean" },
        "node_repl_auto_review_required": { "type": "boolean" },
        "node_repl_disabled": { "type": "boolean" },
        "tool_mode": { "type": ["string", "null"] },
        "model_specialty": { "type": "string" },
        "multi_agent_version": { "type": "string" },
        "multi_agent_reasoning_effort": { "type": "string" }
      },
      "anyOf": [
        { "required": ["base_instructions"] },
        {
          "required": ["model_messages"],
          "properties": {
            "model_messages": {
              "required": ["instructions_template"]
            }
          }
        }
      ],
      "additionalProperties": true
    },
    "reasoning_level": {
      "type": "object",
      "required": ["effort", "description"],
      "properties": {
        "effort": { "type": "string" },
        "description": { "type": "string" }
      },
      "additionalProperties": true
    },
    "service_tier": {
      "type": "object",
      "required": ["id", "name", "description"],
      "properties": {
        "id": { "type": "string" },
        "name": { "type": "string" },
        "description": { "type": "string" }
      },
      "additionalProperties": true
    },
    "upgrade": {
      "type": "object",
      "required": ["model", "migration_markdown"],
      "properties": {
        "model": { "type": "string" },
        "migration_markdown": { "type": "string" },
        "retirement_at": { "type": "string" }
      },
      "additionalProperties": true
    },
    "truncation_policy": {
      "type": "object",
      "required": ["mode", "limit"],
      "properties": {
        "mode": { "type": "string" },
        "limit": { "type": "integer" }
      },
      "additionalProperties": true
    },
    "model_messages": {
      "type": "object",
      "properties": {
        "instructions_template": { "type": "string" },
        "instructions_variables": { "type": ["object", "null"] },
        "approvals": { "type": ["object", "null"] },
        "collaboration_modes": { "type": ["object", "null"] },
        "auto_review": { "type": ["object", "null"] },
        "permissions": { "type": ["object", "null"] },
        "multi_agent": { "type": ["object", "null"] },
        "token_budget": { "type": ["object", "null"] },
        "persistent_instructions": { "type": ["string", "null"] },
        "guardian_v2": { "type": ["object", "null"] },
        "confirmation_policies": { "type": ["object", "null"] }
      },
      "additionalProperties": true
    }
  }
}
```

The parser accepted a model with only the ten required fields above plus
`base_instructions`. It rejected entries missing both instruction sources. It
also rejected reasoning-level objects without either `effort` or `description`,
and truncation policies without either `mode` or `limit`.

## Field guide

The meanings below are inferred from field names, rendered defaults, and the
bundled entries. Only `model_catalog_json` itself is publicly documented.

| Field                              | Observed purpose and values                                                                                                                            |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `slug`                             | Provider-facing model identifier and `/model` selection value.                                                                                         |
| `display_name`                     | Human-readable picker label.                                                                                                                           |
| `description`                      | Picker description; defaults to `null`.                                                                                                                |
| `default_reasoning_level`          | Initial reasoning choice; bundled values are `low` and `medium`.                                                                                       |
| `supported_reasoning_levels`       | Picker choices. Each item requires `effort` and `description`.                                                                                         |
| `shell_type`                       | Shell tool protocol. Every bundled entry uses `unified_exec`.                                                                                          |
| `visibility`                       | Picker visibility; observed values are `list` and `hide`.                                                                                              |
| `supported_in_api`                 | Whether Codex treats the model as API-capable. All bundled entries use `true`.                                                                         |
| `priority`                         | Numeric ordering used by model selection. Lower values appear earlier.                                                                                 |
| `base_instructions`                | Complete base prompt. May replace `model_messages.instructions_template`.                                                                              |
| `model_messages`                   | Template and optional feature-specific instruction fragments.                                                                                          |
| `include_*_usage_instructions`     | Whether Codex injects Skills, Plugin, or Apps usage guidance.                                                                                          |
| `support_verbosity`                | Required flag indicating verbosity support.                                                                                                            |
| `default_verbosity`                | Bundled values are `low`, `medium`, and `high`.                                                                                                        |
| `apply_patch_tool_type`            | Patch protocol. Every bundled entry explicitly uses `freeform`; omission defaults to `null`.                                                           |
| `web_search_tool_type`             | Search protocol; observed values are `text` and `text_and_image`.                                                                                      |
| `truncation_policy`                | Per-tool-output truncation shape. Observed modes are `tokens` and `bytes`, with limit `10000`. This is not the conversation auto-compaction threshold. |
| `context_window`                   | Model context size recorded in the catalog. Optional to parse, but should be explicit for local models.                                                |
| `max_context_window`               | Maximum selectable or supported context recorded for the model.                                                                                        |
| `effective_context_window_percent` | Safety percentage used by Codex's context accounting; every bundled entry uses `95`. This field is not publicly documented.                            |
| `comp_hash`                        | Opaque compatibility/version value. It is omitted from one bundled entry and should not be invented for local models without evidence.                 |
| `experimental_supported_tools`     | Required array of experimental tool identifiers; use `[]` for a conservative local entry.                                                              |
| `input_modalities`                 | Observed modalities are `text` and `image`. Omission rendered as both.                                                                                 |
| `supports_search_tool`             | Model capability flag; this does not itself enable search in the session.                                                                              |
| `supports_experimental_context`    | Enables Codex's experimental larger-context behavior.                                                                                                  |
| `use_responses_lite`               | Selects the newer lightweight Responses transport behavior for compatible models.                                                                      |
| `tool_mode`                        | `code_mode_only` for newer bundled models; omission or `null` produces the direct-tool path.                                                           |
| `multi_agent_version`              | Observed values are `v1` and `v2`; absent for some direct-tool models.                                                                                 |
| `upgrade`                          | Optional retirement/migration metadata for a superseded model.                                                                                         |

Observed reasoning efforts are `low`, `medium`, `high`, `xhigh`, `max`, and
`ultra`. Observed service-tier ids are `priority` and `ultrafast`. These are
observations, not promises that arbitrary providers or local models implement
the corresponding behavior.

## Bundled OpenAI models

Instruction sizes are character counts from the unrendered bundled strings.
`direct` below means `tool_mode` was absent rather than the literal string
`direct`.

| Slug                       | Picker | Context / max | Efforts                              | Tool mode      | Search       | Base / template chars |
| -------------------------- | ------ | ------------: | ------------------------------------ | -------------- | ------------ | --------------------: |
| `gpt-6-astra`              | list   |   272K / 872K | low, medium, high, xhigh, max, ultra | code-mode-only | text + image |       21,261 / 21,261 |
| `gpt-5.6-sol`              | list   |   272K / 872K | low, medium, high, xhigh, max, ultra | code-mode-only | text + image |       17,730 / 17,730 |
| `gpt-5.6-terra`            | list   |   272K / 872K | low, medium, high, xhigh, max, ultra | code-mode-only | text + image |       17,730 / 17,730 |
| `gpt-5.6-luna`             | list   |   272K / 872K | low, medium, high, xhigh, max        | code-mode-only | text + image |       17,730 / 17,730 |
| `gpt-daybreak-blue-latest` | hidden |   272K / 872K | low, medium, high, xhigh, max, ultra | code-mode-only | text + image |       17,298 / 17,298 |
| `gpt-daybreak-red-latest`  | hidden |   372K / 372K | low, medium, high, xhigh, max, ultra | code-mode-only | text + image |       17,297 / 17,297 |
| `gpt-5.5`                  | list   |   272K / 272K | low, medium, high, xhigh             | direct         | text + image |       19,737 / 19,754 |
| `gpt-5.4`                  | hidden | 272K / 1,000K | low, medium, high, xhigh             | direct         | text + image |       12,879 / 12,896 |
| `gpt-5.4-mini`             | hidden |   272K / 272K | low, medium, high, xhigh             | direct         | text + image |       11,097 / 11,114 |
| `gpt-5.2`                  | list   |   272K / 272K | low, medium, high, xhigh             | direct         | text         |       21,544 / 21,544 |
| `codex-auto-review`        | hidden |   272K / 872K | low, medium, high, xhigh, max        | code-mode-only | text + image |       17,298 / 17,298 |

All bundled entries use `unified_exec`, `freeform` apply-patch, a 95 percent
effective context window, and a truncation limit of 10,000. Ten entries use
token truncation; `gpt-5.2` uses byte truncation.

## Example: GPT-5.6 Sol

This is the bundled metadata with the 17,730-character instruction strings
omitted. The reproduction command above prints their exact contents. Omission
markers make this an explanatory view, not a catalog entry to load directly.

```json
{
  "slug": "gpt-5.6-sol",
  "display_name": "GPT-5.6-Sol",
  "description": "Latest frontier agentic coding model.",
  "default_reasoning_level": "low",
  "supported_reasoning_levels": [
    { "effort": "low", "description": "Fast responses with lighter reasoning" },
    {
      "effort": "medium",
      "description": "Balances speed and reasoning depth for everyday tasks"
    },
    {
      "effort": "high",
      "description": "Greater reasoning depth for complex problems"
    },
    {
      "effort": "xhigh",
      "description": "Extra high reasoning depth for complex problems"
    },
    {
      "effort": "max",
      "description": "Maximum reasoning depth for the hardest problems"
    },
    {
      "effort": "ultra",
      "description": "Maximum reasoning with automatic task delegation"
    }
  ],
  "shell_type": "unified_exec",
  "visibility": "list",
  "supported_in_api": true,
  "priority": 6,
  "additional_speed_tiers": ["fast"],
  "service_tiers": [
    {
      "id": "priority",
      "name": "Fast",
      "description": "1.5x speed, increased usage"
    },
    {
      "id": "ultrafast",
      "name": "Ultrafast",
      "description": "The fastest available responses for latency-sensitive work."
    }
  ],
  "availability_nux": null,
  "upgrade": null,
  "model_messages": {
    "instructions_template": "<17,730 characters omitted>",
    "instructions_variables": null,
    "approvals": null,
    "collaboration_modes": null,
    "auto_review": null,
    "permissions": null,
    "multi_agent": null,
    "token_budget": "<nested token-budget configuration omitted>"
  },
  "include_skills_usage_instructions": false,
  "include_plugin_usage_instructions": true,
  "include_apps_usage_instructions": true,
  "default_reasoning_summary": "none",
  "support_verbosity": true,
  "default_verbosity": "low",
  "apply_patch_tool_type": "freeform",
  "web_search_tool_type": "text_and_image",
  "truncation_policy": { "mode": "tokens", "limit": 10000 },
  "supports_image_detail_original": true,
  "context_window": 272000,
  "max_context_window": 872000,
  "comp_hash": "3000",
  "effective_context_window_percent": 95,
  "experimental_supported_tools": [],
  "input_modalities": ["text", "image"],
  "supports_search_tool": true,
  "supports_experimental_context": false,
  "use_responses_lite": true,
  "node_repl_auto_review_required": false,
  "node_repl_disabled": false,
  "tool_mode": "code_mode_only",
  "multi_agent_version": "v2"
}
```

## Example: GPT-5.5

GPT-5.5 demonstrates the older direct-tool path and instruction variables.
The instruction omission markers make this an explanatory view rather than a
loadable entry.

```json
{
  "slug": "gpt-5.5",
  "display_name": "GPT-5.5",
  "description": "Frontier model for complex coding, research, and real-world work.",
  "default_reasoning_level": "medium",
  "supported_reasoning_levels": [
    { "effort": "low", "description": "Fast responses with lighter reasoning" },
    {
      "effort": "medium",
      "description": "Balances speed and reasoning depth for everyday tasks"
    },
    {
      "effort": "high",
      "description": "Greater reasoning depth for complex problems"
    },
    {
      "effort": "xhigh",
      "description": "Extra high reasoning depth for complex problems"
    }
  ],
  "shell_type": "unified_exec",
  "visibility": "list",
  "supported_in_api": true,
  "priority": 12,
  "additional_speed_tiers": ["fast"],
  "service_tiers": [
    {
      "id": "priority",
      "name": "Fast",
      "description": "1.5x speed, increased usage"
    }
  ],
  "availability_nux": null,
  "upgrade": null,
  "model_messages": {
    "instructions_template": "<19,754 characters omitted>",
    "instructions_variables": {
      "personality_default": "",
      "personality_friendly": "<personality instructions omitted>",
      "personality_pragmatic": "<personality instructions omitted>"
    },
    "approvals": null,
    "collaboration_modes": null,
    "auto_review": null,
    "permissions": null,
    "multi_agent": null
  },
  "include_skills_usage_instructions": true,
  "include_plugin_usage_instructions": true,
  "include_apps_usage_instructions": true,
  "default_reasoning_summary": "none",
  "support_verbosity": true,
  "default_verbosity": "low",
  "apply_patch_tool_type": "freeform",
  "web_search_tool_type": "text_and_image",
  "truncation_policy": { "mode": "tokens", "limit": 10000 },
  "supports_image_detail_original": true,
  "context_window": 272000,
  "max_context_window": 272000,
  "comp_hash": "2911",
  "effective_context_window_percent": 95,
  "experimental_supported_tools": [],
  "input_modalities": ["text", "image"],
  "supports_search_tool": true,
  "supports_experimental_context": false,
  "use_responses_lite": false,
  "node_repl_auto_review_required": false,
  "node_repl_disabled": false
}
```

## Parser defaults relevant to local models

Starting from a bundled GPT-5.5 entry, the 0.154.0 parser accepted deletion of
every field except the ten fields marked required in the schema. Supplying
`base_instructions` without `model_messages` produced these relevant defaults:

| Field                               | Rendered default when omitted          |
| ----------------------------------- | -------------------------------------- |
| `description`                       | `null`                                 |
| `additional_speed_tiers`            | `[]`                                   |
| `service_tiers`                     | `[]`                                   |
| `upgrade`                           | `null`                                 |
| `include_skills_usage_instructions` | `false`                                |
| `include_plugin_usage_instructions` | `false`                                |
| `include_apps_usage_instructions`   | `true`                                 |
| `default_reasoning_summary`         | `auto`                                 |
| `default_verbosity`                 | `null`                                 |
| `apply_patch_tool_type`             | `null`                                 |
| `web_search_tool_type`              | `text`                                 |
| `effective_context_window_percent`  | `95`                                   |
| `input_modalities`                  | `["text", "image"]`                    |
| `supports_search_tool`              | `false`                                |
| `supports_experimental_context`     | `false`                                |
| `use_responses_lite`                | `false`                                |
| `node_repl_auto_review_required`    | `false`                                |
| `node_repl_disabled`                | `false`                                |
| `tool_mode`                         | absent, selecting the direct-tool path |

For this repository's local entries, explicitly set fields that affect context,
tool exposure, transport, or model capability instead of relying on these
defaults. In particular, omitting `apply_patch_tool_type` stops requesting the dedicated patch protocol, while omitting `include_apps_usage_instructions` defaults it to `true`. Declaring the field does not prove that a local provider and model expose the freeform tool end to end; the tested LM Studio/Qwen path did not. See [the local patch compatibility notes](../apply-patch.md).

## Validate a custom catalog

```zsh
codex \
  -c 'model_catalog_json="/absolute/path/to/catalog.json"' \
  debug models |
  jq
```

Validation proves that the installed parser accepts the shape. It does not
prove that a local provider or model implements every declared capability. Tool
calling, compaction, reasoning parameters, image input, and in-session model
switching still require runtime smoke tests.

## Official documentation

- [Codex configuration reference](https://learn.chatgpt.com/docs/config-file/config-reference)
- [Codex advanced configuration](https://learn.chatgpt.com/docs/config-file/config-advanced)

The official pages document how to select `model_catalog_json`, but not the
entry schema recorded above.
