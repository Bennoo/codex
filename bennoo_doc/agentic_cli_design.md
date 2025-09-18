# Agentic CLI Architecture Guide

## Overview
Codex's Rust CLI is a multitool binary that routes user intents to different front-ends while relying on a shared agent runtime. The top-level `codex-rs/cli` crate defines the `MultitoolCli` parser and dispatches to either the interactive TUI, the headless executor, or utility subcommands, forwarding configuration overrides before delegating to the selected adapter.【F:codex-rs/cli/src/main.rs†L25-L195】 Both the terminal UI (`codex-rs/tui`) and the headless `codex-rs/exec` driver embed the same `codex-core` agent crate, which owns the conversation lifecycle, model streaming, and tool orchestration.

## CLI entrypoint crate (`codex-rs/cli`)
* `main.rs` wires clap's parser, provides subcommands such as `exec`, `proto`, and `apply`, and defaults to launching the interactive UI when no subcommand is specified.【F:codex-rs/cli/src/main.rs†L25-L195】
* `proto.rs`, `login.rs`, and `debug_sandbox.rs` layer CLI UX over shared services (MCP streams, auth, sandboxes). These modules do not contain agent logic; instead they collect arguments and hand off to `codex-core` or helper crates exposed in the workspace.

## Interactive adapter (`codex-rs/tui`)
The TUI spawns the agent from its chat widget. `spawn_agent` creates a `ConversationManager`, initializes a new conversation, forwards the initial `SessionConfigured` event to the UI, and runs two asynchronous loops: one forwarding UI operations (`Op`) into the agent and another streaming events back into Ratatui widgets.【F:codex-rs/tui/src/chatwidget/agent.rs†L14-L95】 When resuming an archived conversation, `spawn_agent_from_existing` reuses a stored `CodexConversation` with the same routing logic.【F:codex-rs/tui/src/chatwidget/agent.rs†L64-L99】

## Headless executor (`codex-rs/exec`)
The non-interactive `codex exec` path configures logging, resolves sandbox mode, merges CLI overrides, and loads a `codex-core::Config`. It then creates a `ConversationManager`, optionally resumes a rollout, spawns the agent, and relays events through an internal channel while the user prompt and attachments are submitted as `Op::UserInput`. The executor waits for task completion events, reacts to shutdown requests, and persists resume metadata for later turns.【F:codex-rs/exec/src/lib.rs†L201-L309】

## Core agent crate (`codex-core`)
`codex-core` is the heart of the agentic design. Its `lib.rs` re-exports a large surface area so adapters can consume configuration, protocol types, and helpers without touching internal modules directly.【F:codex-rs/core/src/lib.rs†L8-L98】 Key building blocks include:

### Configuration and model metadata
`Config` captures the model selection, approval policy, sandbox policy, shell environment settings, MCP servers, and notification hooks that govern each session. It also encodes whether to hide or expose the agent's reasoning stream, ensuring front-ends can toggle detailed traces.【F:codex-rs/core/src/config.rs†L51-L140】 The configuration API merges disk state, CLI overrides, and dynamic turn overrides so adapters can adjust parameters (for example, switching models mid-session).

### Conversation lifecycle
`ConversationManager` constructs new conversations, handles resumes and forks via the rollout recorder, and validates that the first emitted event is the `SessionConfigured` handshake before returning a `CodexConversation` handle. It keeps an `Arc<RwLock<HashMap>>` of active sessions so UIs can look up or remove conversations safely.【F:codex-rs/core/src/conversation_manager.rs†L23-L196】

`Codex` itself wraps submission and event channels. `Codex::spawn` builds the initial configuration, creates a `Session`, and launches the asynchronous `submission_loop` task. The public API exposes `submit`, which auto-generates submission IDs, and `next_event`, which streams processed `Event` messages back to callers.【F:codex-rs/core/src/codex.rs†L180-L262】

### Session state and persistence
Each `Session` holds the active conversation ID, outbound event sender, MCP connection manager, exec session managers, optional rollout recorder, notification hooks, and flags controlling reasoning visibility.【F:codex-rs/core/src/codex.rs†L280-L301】 Events are persisted to the rollout log before being forwarded to clients via `send_event`, ensuring replay and resume support.【F:codex-rs/core/src/codex.rs†L584-L591】 The session also manages approval futures, pending input, and contextual metadata across turns.

### Task orchestration
`AgentTask` represents an asynchronous turn runner. It spawns `run_task` on the Tokio runtime, tracks whether the task is regular, review, or compaction work, and allows cancellation with appropriate `TurnAborted` events.【F:codex-rs/core/src/codex.rs†L1081-L1172】 `run_task` drives the core agent loop: it records user input, maintains per-turn history, streams the model's `ResponseItem` values, executes tool calls, records assistant messages, and emits task lifecycle events until completion or interruption.【F:codex-rs/core/src/codex.rs†L1605-L1756】

### Mapping model output to UI events
As the model emits `ResponseItem` structures (from the Responses API or chat completions), `event_mapping::map_response_item_to_event_messages` converts them into `EventMsg` variants understood by the CLI front-ends, optionally filtering raw reasoning traces based on configuration.【F:codex-rs/core/src/event_mapping.rs†L1-L123】

### Tooling layers
`openai_tools` computes a `ToolsConfig` per session or turn, enabling or disabling plan, apply-patch, web search, view image, streamable shell, and unified exec tools based on model family and approval policy.【F:codex-rs/core/src/openai_tools.rs†L60-L134】 When the agent invokes the shell, `exec::process_exec_tool_call` coordinates sandbox selection, streaming stdout, timeout handling, and exit-code interpretation across macOS Seatbelt, Linux Landlock, or direct execution.【F:codex-rs/core/src/exec.rs†L1-L183】 `apply_patch` performs safety assessment, surfaces approval prompts, and either delegates to exec or returns structured failure payloads to the model.【F:codex-rs/core/src/apply_patch.rs†L1-L100】 MCP integration is handled by `McpConnectionManager`, which spawns configured servers, tracks tool registries, and stores clients, and by `mcp_tool_call::handle_mcp_tool_call`, which emits begin/end events and feeds tool outputs back to the model as function call results.【F:codex-rs/core/src/mcp_connection_manager.rs†L92-L217】【F:codex-rs/core/src/mcp_tool_call.rs†L15-L74】 Together these modules implement the agent's ability to plan, request approvals, run commands, apply patches, and call external MCP tools.

## Protocol re-export (`codex-rs/protocol`)
Rather than redefining protocol structures, `codex-core` re-exports the `codex-protocol` crate so consumers can import types like `Op`, `EventMsg`, and `InitialHistory` through `codex_core::protocol`. This keeps the CLI adapters in sync with the protocol schema maintained in the dedicated crate.【F:codex-rs/core/src/lib.rs†L47-L87】

## End-to-end flow
1. The CLI parses arguments and delegates to either the TUI or executor adapter.【F:codex-rs/cli/src/main.rs†L166-L195】
2. The adapter loads configuration, prepares a `ConversationManager`, and requests a new or resumed conversation, receiving a `CodexConversation` and the initial session event.【F:codex-rs/tui/src/chatwidget/agent.rs†L14-L58】【F:codex-rs/exec/src/lib.rs†L201-L226】
3. User actions are translated into protocol `Op`s, which the adapter forwards to `Codex::submit`; `Codex` queues them for the `submission_loop` running inside the core crate.【F:codex-rs/core/src/codex.rs†L180-L262】【F:codex-rs/core/src/codex.rs†L1177-L1389】
4. `AgentTask` executes turns, invoking tools (`exec`, `apply_patch`, MCP) as requested, while `event_mapping` and `send_event` persist and broadcast resulting `EventMsg`s back to the adapter.【F:codex-rs/core/src/codex.rs†L1081-L1756】【F:codex-rs/core/src/event_mapping.rs†L1-L123】
5. The adapter renders or prints these events and reacts to completion or shutdown signals (e.g., submitting `Op::Shutdown` when the exec driver finishes).【F:codex-rs/exec/src/lib.rs†L228-L305】

With this separation, front-end crates focus on UX while `codex-core` concentrates agent state, tool plumbing, and protocol compliance. Adding new agent behaviors typically means extending `codex-core` (e.g., new tools, approval policies) and wiring minimal glue in the adapters to expose configuration toggles.
