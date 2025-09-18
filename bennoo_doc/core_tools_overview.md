# codex-core Tooling Overview

This guide summarizes every tool that the `codex-core` crate makes available to
the model, explains how those tool definitions are assembled into OpenAI
Requests, and documents the runtime code paths that execute tool invocations.

## Where tool definitions live

The core crate models each OpenAI tool as an `OpenAiTool`, which can be a
function, the built-in `local_shell`/`web_search` primitives, or a custom
freeform grammar.【F:codex-rs/core/src/openai_tools.rs†L41-L55】 `ToolsConfig`
chooses which tool variants to expose for the current model family and runtime
flags (plan tool, apply_patch flavour, unified exec, web search, image
attachment, etc.).【F:codex-rs/core/src/openai_tools.rs†L66-L135】

### Shell and execution tools

* `shell` (standard function call) – defined by `create_shell_tool`, which
  accepts `command`, `workdir`, and `timeout_ms` arguments.【F:codex-rs/core/src/openai_tools.rs†L173-L205】
* `shell` (with explicit sandbox request fields) – emitted when approval policy
  requires the model to request elevated privileges, via
  `create_shell_tool_for_sandbox`. It augments the standard schema with
  `with_escalated_permissions` and `justification`.【F:codex-rs/core/src/openai_tools.rs†L254-L301】
* `local_shell` – exposed by `ToolsConfig` when the model family natively
  supports the built-in shell tool.【F:codex-rs/core/src/openai_tools.rs†L103-L128】【F:codex-rs/core/src/openai_tools.rs†L543-L548】
* Streamable shell pair – `exec_command` and `write_stdin` are registered when
  streamable shell support is requested.【F:codex-rs/core/src/openai_tools.rs†L549-L555】 The
  request/response schema for each tool lives in `exec_command::responses_api`.
  `exec_command` executes a command with streaming output, while `write_stdin`
  sends characters to an existing session.【F:codex-rs/core/src/exec_command/responses_api.rs†L9-L64】【F:codex-rs/core/src/exec_command/responses_api.rs†L66-L111】
* `unified_exec` – an experimental PTY wrapper that multiplexes command spawning
  and stdin writes through a single tool.【F:codex-rs/core/src/openai_tools.rs†L207-L252】

### File editing tools

* `apply_patch` freeform custom tool – backed by a Lark grammar so GPT-5 models
  can emit structured patches.【F:codex-rs/core/src/tool_apply_patch.rs†L19-L33】 The grammar is
  embedded from `tool_apply_patch.lark` (same directory).
* `apply_patch` JSON function – alternative schema for models that do not
  support freeform grammars.【F:codex-rs/core/src/tool_apply_patch.rs†L35-L98】
* Both variants are selected by `ToolsConfig` (model defaults or
  `include_apply_patch_tool`).【F:codex-rs/core/src/openai_tools.rs†L114-L133】【F:codex-rs/core/src/openai_tools.rs†L564-L573】

### Planning and context enrichment

* `update_plan` – the structured planning tool (`PLAN_TOOL`) that records the
  agent’s task plan; defined in `plan_tool.rs` with a schema for plan steps and
  statuses.【F:codex-rs/core/src/plan_tool.rs†L18-L66】 `ToolsConfig` includes it when the
  session asks for planning support.【F:codex-rs/core/src/openai_tools.rs†L560-L562】
* `view_image` – allows the agent to attach a local filesystem image path to the
  turn context.【F:codex-rs/core/src/openai_tools.rs†L303-L324】
* `web_search` – registered when a session wants web search escalation.【F:codex-rs/core/src/openai_tools.rs†L575-L577】

### Model Context Protocol (MCP) tools

When MCP servers are connected, their tools are pulled into the same
`OpenAiTool` list. `get_openai_tools` sorts the qualified MCP tool map and wraps
each remote tool schema via `mcp_tool_to_openai_tool`, which sanitizes incoming
JSON Schema so that it fits the limited enum used by OpenAI tool definitions.【F:codex-rs/core/src/openai_tools.rs†L583-L598】【F:codex-rs/core/src/openai_tools.rs†L378-L411】
`McpConnectionManager` is responsible for discovering server tools, assigning
fully qualified names, and exposing helpers to call them later.【F:codex-rs/core/src/mcp_connection_manager.rs†L231-L320】

## How tools are attached to model requests

Callers build a `ToolsConfig` from runtime context (model family, approval
policy, feature flags) and then ask `get_openai_tools` for the definitive list
of `OpenAiTool` objects to expose.【F:codex-rs/core/src/openai_tools.rs†L527-L599】 That list is
converted into the API-specific payload by `create_tools_json_for_responses_api`
(experimental Responses API) or the chat-completions equivalent, which strips
non-function tools that the legacy endpoint cannot accept.【F:codex-rs/core/src/openai_tools.rs†L332-L376】

* Responses API usage – `ModelClient::stream_responses` calls
  `create_tools_json_for_responses_api` while assembling the POST body (tools,
  instructions, prompt cache key, etc.).【F:codex-rs/core/src/client.rs†L157-L224】
* Chat Completions usage – `stream_chat_completions` mirrors the same step but
  routes through `create_tools_json_for_chat_completions_api` before sending the
  SSE request.【F:codex-rs/core/src/chat_completions.rs†L240-L306】

## Tool invocation and execution flow

The streaming handler `handle_response_item` inspects each `ResponseItem` from
OpenAI. It dispatches function calls, local shell events, or custom tool calls
into specialized executors while forwarding regular assistant messages and
reasoning deltas to the UI.【F:codex-rs/core/src/codex.rs†L2211-L2333】

### Function-style tools

* `handle_function_call` matches the tool name, parses JSON arguments, and
  invokes the appropriate helper. It covers shell/unified exec, plan updates,
  view_image, apply_patch, streamable shell (`exec_command`/`write_stdin`), and
  routes unknown names through the MCP dispatcher.【F:codex-rs/core/src/codex.rs†L2401-L2592】
* Shell invocations reuse `parse_container_exec_arguments` to convert the JSON
  payload into `ExecParams` (command, cwd, timeout, environment).【F:codex-rs/core/src/codex.rs†L2647-L2678】
* `handle_unified_exec_tool_call` packages PTY requests for the unified exec
  manager and serializes its result back to the model.【F:codex-rs/core/src/codex.rs†L2335-L2399】
* `handle_update_plan` (from `plan_tool.rs`) validates the structured plan and
  emits a `PlanUpdate` event before confirming success to the model.【F:codex-rs/core/src/plan_tool.rs†L44-L81】
* `handle_mcp_tool_call` wraps calls to external MCP servers, emitting begin/end
  events and returning their JSON payloads to the assistant turn.【F:codex-rs/core/src/mcp_tool_call.rs†L15-L71】

### Custom tools

Custom (non-function) tool calls – currently the freeform `apply_patch`
variant – go through `handle_custom_tool_call`, which reuses the exec pipeline
and converts the eventual function-style response into a custom tool output so
OpenAI can continue the turn with the right call ID.【F:codex-rs/core/src/codex.rs†L2595-L2644】

### Executing shell and patch commands

`handle_container_exec_with_params` is the common execution engine. It:
1. Detects whether the request is an `apply_patch` invocation and either applies
   the patch internally or rewrites the command to call the CLI helper with the
   correct arguments.【F:codex-rs/core/src/codex.rs†L2706-L2788】
2. Performs safety checks, potentially prompting the user for approval and
   selecting the sandbox strategy.【F:codex-rs/core/src/codex.rs†L2789-L2848】
3. Delegates to the sandboxed executor while tracking stdout/stderr for the turn
   diff tracker (rest of the function continues beyond the excerpt above).

Supporting helpers translate the request-specified working directory into an
absolute path (`to_exec_params`) and optionally adjust commands for the user’s
shell (`maybe_translate_shell_command`).【F:codex-rs/core/src/codex.rs†L2647-L2703】

Together, these modules describe every tool the agent can call, how those tools
are offered to the model, and where the runtime enforces parsing, safety, and
execution of tool invocations.

## ASCII data flow summary

```
┌──────────────────────────────────────────────┐
│ Runtime context (model, flags, approvals)    │
└──────────────────────────────────────────────┘
                    │
                    ▼
        ┌────────────────────────┐
        │ ToolsConfig builder    │
        │ (openai_tools.rs)      │
        └────────────────────────┘
                    │ selects tool variants
                    ▼
      ┌──────────────────────────────────┐
      │ get_openai_tools / MCP merger    │
      │ → Vec<OpenAiTool>                │
      └──────────────────────────────────┘
                    │ serialized by
                    ▼
  ┌─────────────────────────────────────────┐
  │ create_tools_json_* (client/chat APIs)  │
  │ inject tools into OpenAI request body   │
  └─────────────────────────────────────────┘
                    │ stream responses
                    ▼
      ┌────────────────────────────────┐
      │ handle_response_item (codex.rs)│
      └────────────────────────────────┘
          │ function-call         │ custom tool
          ▼                       ▼
┌───────────────────────┐   ┌──────────────────────┐
│ handle_function_call  │   │ handle_custom_tool   │
│ → exec/plan/MCP/etc.  │   │ → apply_patch path   │
└───────────────────────┘   └──────────────────────┘
          │                       │
          ▼                       ▼
  ┌──────────────────────────┐    │
  │ handle_container_exec_*  │◀───┘
  │ → sandbox + command exec │
  └──────────────────────────┘
```
