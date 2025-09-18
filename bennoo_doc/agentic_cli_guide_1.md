# Navigating the Rust Agentic CLI Code

This guide points to the Rust crates and modules that implement Codex's agentic command-line experience. It explains how the interactive CLI is bootstrapped, how agent sessions are created, and which components handle model output, tool calls, and approvals.

## Top-level Architecture

The CLI experience spans three Rust crates in `codex-rs`:

- **`codex-rs/cli`** – parses the user-facing `codex` binary flags and dispatches into the appropriate operating mode (interactive TUI, non-interactive exec, MCP utilities, etc.).【F:codex-rs/cli/src/main.rs†L1-L200】
- **`codex-rs/tui`** – hosts the interactive terminal UI, including the event loop, chat widget, resume picker, approvals, and rendering logic.【F:codex-rs/tui/src/lib.rs†L1-L248】【F:codex-rs/tui/src/app.rs†L1-L157】
- **`codex-rs/core`** – contains the agent runtime: session lifecycle management, task execution, tool orchestration, sandbox policies, and protocol event generation.【F:codex-rs/core/src/lib.rs†L1-L68】【F:codex-rs/core/src/codex.rs†L180-L260】

The CLI crate orchestrates the others: invoking the TUI crate spins up a `ConversationManager` from `codex-core`, which in turn spawns the underlying agent session and streams events back to the UI.

## `codex-rs/cli`: Entry Point and Dispatch

`codex-rs/cli/src/main.rs` defines the `MultitoolCli` parser and a `Subcommand` enum. When no subcommand is provided, it forwards flags to `codex_tui::run_main`; subcommands route to specialized flows such as non-interactive execution, authentication helpers, or protocol tooling.【F:codex-rs/cli/src/main.rs†L25-L200】 The helper `prepend_config_flags` ensures root-level `-c key=value` overrides are merged before calling into the specific mode.【F:codex-rs/cli/src/main.rs†L166-L320】

`lib.rs` exposes shared argument types like `SeatbeltCommand`, `LandlockCommand`, and login helpers that other crates reuse when embedding sandbox or auth management into their CLIs.【F:codex-rs/cli/src/lib.rs†L1-L40】 Although these modules do not host agent logic directly, they provide the hooks the interactive mode uses to configure sandbox or resume behavior before the agent starts.

## `codex-rs/tui`: Interactive Agent Shell

`run_main` in `codex-rs/tui/src/lib.rs` is the bridge between CLI arguments and the UI runtime. It normalizes sandbox/approval flags, loads configuration, prepares logging, and finally calls `run_ratatui_app` to launch the interface.【F:codex-rs/tui/src/lib.rs†L85-L252】 The module tree declared in the same file maps each feature—chat widget, approvals, file search, plan updates, session logs, etc.—to a dedicated submodule.【F:codex-rs/tui/src/lib.rs†L31-L69】

`App::run` (in `app.rs`) constructs the agent-facing subsystems: it instantiates an async channel for UI events, creates a shared `ConversationManager`, and builds a `ChatWidget` either from scratch or by resuming an archived rollout.【F:codex-rs/tui/src/app.rs†L64-L140】 The `App` event loop multiplexes TUI input and agent events, handing each to the chat widget for rendering or follow-up actions.【F:codex-rs/tui/src/app.rs†L142-L157】

### Chat Widget responsibilities

`chatwidget.rs` owns the high-level agent interaction surface. It keeps the outbound `UnboundedSender<Op>` used to submit protocol operations, manages history rendering, tracks running exec commands, and accumulates reasoning snippets for transcript logging.【F:codex-rs/tui/src/chatwidget.rs†L74-L139】 Event handlers stream assistant messages, accumulate reasoning sections, mark task boundaries, and surface plan updates, approvals, tool invocations, or errors to the UI.【F:codex-rs/tui/src/chatwidget.rs†L194-L372】 Constructors wire the widget to a new or existing conversation by calling the helper functions in `chatwidget/agent.rs` to spawn the async agent loops.【F:codex-rs/tui/src/chatwidget.rs†L641-L745】

`chatwidget/agent.rs` is the glue between the UI and `codex-core`. It opens a new conversation through `ConversationManager::new_conversation`, forwards the initial `SessionConfigured` event to the UI, and then spins two asynchronous tasks: one that relays UI `Op`s into the agent and another that streams `Event`s back to the chat widget.【F:codex-rs/tui/src/chatwidget/agent.rs†L14-L99】 The companion function handles the resumption path by reusing an existing `CodexConversation` handle.【F:codex-rs/tui/src/chatwidget/agent.rs†L64-L99】

## `codex-rs/core`: Agent Runtime and Tools

`codex_core::ConversationManager` creates, resumes, and tracks `CodexConversation` instances. It enforces that the first event emitted is `SessionConfigured`, stores the conversations in memory, and exposes helpers for forking or resuming sessions from rollout logs.【F:codex-rs/core/src/conversation_manager.rs†L23-L160】 Each `CodexConversation` is just a thin wrapper that forwards `submit` and `next_event` calls to the underlying `Codex` handle.【F:codex-rs/core/src/codex_conversation.rs†L7-L29】

`Codex::spawn` sets up the agent session. It loads user/project instructions, builds the initial `Session` and `TurnContext`, and launches the `submission_loop`, which processes incoming `Op`s until shutdown.【F:codex-rs/core/src/codex.rs†L180-L232】 The same module defines `AgentTask`, representing a single multi-turn interaction, and the logic for interrupting, reviewing, or compacting ongoing work.【F:codex-rs/core/src/codex.rs†L1081-L1174】 The `submission_loop` matches each incoming `Op`—overrides, user turns, approvals, tool listings, history lookups, shutdowns—and performs the corresponding side effects before emitting protocol events back to the UI.【F:codex-rs/core/src/codex.rs†L1177-L1527】 Supporting routines such as `spawn_review_thread` and `run_task` handle specialized flows like review mode and tool invocation loops.【F:codex-rs/core/src/codex.rs†L1529-L1640】

The crate root (`core/src/lib.rs`) enumerates additional modules you can explore when extending agent behavior: execution sandboxes (`exec`, `landlock`, `seatbelt`), tool integrations (`plan_tool`, `tool_apply_patch`, `exec_command`), configuration management, rollout recording, and protocol re-exports that make `codex_core::protocol::*` available across the workspace.【F:codex-rs/core/src/lib.rs†L1-L68】 These modules plug into the agent loop through the tool orchestration inside `codex.rs` and the `ToolsConfig` builders used during turn setup.【F:codex-rs/core/src/codex.rs†L1192-L1342】

## Putting It Together: Flow of Control

1. **CLI parsing:** `codex` parses arguments and, in interactive mode, calls `codex_tui::run_main`, passing shared config overrides.【F:codex-rs/cli/src/main.rs†L166-L320】
2. **TUI bootstrap:** `run_main` resolves configuration, logging, onboarding, and resume options, then hands control to `App::run` with the chosen prompt/images and active profile.【F:codex-rs/tui/src/lib.rs†L85-L429】
3. **Conversation setup:** `App::run` creates a `ConversationManager`, builds a `ChatWidget`, and the widget spawns agent tasks to open a new conversation and listen for protocol events.【F:codex-rs/tui/src/app.rs†L64-L140】【F:codex-rs/tui/src/chatwidget.rs†L641-L745】【F:codex-rs/tui/src/chatwidget/agent.rs†L14-L99】
4. **Agent runtime:** `ConversationManager` wraps the `Codex::spawn`ed session, while the `submission_loop` processes incoming `Op`s (user turns, approvals, resume hooks) and emits `Event`s such as task starts, reasoning deltas, tool calls, and completions.【F:codex-rs/core/src/conversation_manager.rs†L52-L160】【F:codex-rs/core/src/codex.rs†L180-L1527】
5. **UI rendering & follow-up:** The chat widget receives those events, updates the transcript, queues approvals, tracks running commands, and sends follow-up `Op`s (e.g., approval decisions, history requests) via its `codex_op_tx` channel.【F:codex-rs/tui/src/chatwidget.rs†L194-L372】 The cycle repeats for each task until the session ends.

With this map, you can trace any interactive feature—from the CLI flag that enables it, through the UI module that surfaces it, down to the agent runtime that executes it—and identify the Rust files that need to be touched when evolving the agentic design of the Codex CLI.
