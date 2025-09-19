# TUI ↔ core integration guide

This document describes how the `codex-tui` crate talks to `codex-core`. It focuses on the
concrete touch points between both crates so that future work can reuse the existing
plumbing instead of reinventing it.

## 1. Architectural overview
- `codex-tui` owns terminal UI state (ratatui widgets, input handling, overlays).
- `codex-core` owns authentication, configuration, conversation orchestration, the
  execution sandbox, and re-exports the shared protocol types from `codex-protocol`.
- Communication happens through two async channels per session:
  - `Op` submissions produced by the TUI and consumed by `codex-core`.
  - `Event` messages emitted by `codex-core` and rendered by the TUI.
- `codex_core::ConversationManager` constructs conversations and hands back an
  `Arc<CodexConversation>` that exposes `submit(Op)` and `next_event()`.
- The `App` type in `codex-tui` wires the UI event loop to these channels via
  `AppEvent` messages.

```
          +-------------+                     +-------------------+
          |  codex-tui  |                     |    codex-core     |
          |             |  Op (unbounded) --> |  CodexConversation|
  user ↔  |  ChatWidget |                     | + ConversationMgr |
          |             | <-- Event (async)   |                   |
          +-------------+                     +-------------------+
```

## 2. Bootstrap sequence (`codex-rs/tui/src/lib.rs`)
1. **CLI parsing & overrides** – `run_main` merges `Cli` flags with
   `codex_core::config::ConfigOverrides` and `CliConfigOverrides` to construct an
   effective `Config`. It calls `Config::load_with_cli_overrides` and
   `load_config_as_toml_with_cli_overrides` to hydrate both the runtime config and the
   serialized TOML view.
2. **Trust & sandbox defaults** – `determine_repo_trust_state` checks whether CLI,
   profile, or persisted settings already declare sandbox/approval policies. If not, it
   consults `config_toml.is_cwd_trusted()` and may set defaults to
   `AskForApproval::OnRequest` and a workspace-write sandbox.
3. **Internal storage** – `codex_core::internal_storage::InternalStorage::load` is read to
   gate once-per-user prompts (for example the GPT-5 Codex upgrade banner).
4. **Logging/tracing** – `codex_core::config::log_dir` determines the log directory. The
   TUI opens `codex-tui.log` there and installs a `tracing_subscriber` layer so core logs
   and UI logs land in the same file.
5. **OSS bootstrap** – When `--oss` is active the TUI calls `codex_ollama::ensure_oss_ready`
   after config has been constructed.
6. **Onboarding** – `AuthManager::shared` and `get_login_status` decide whether to render
   onboarding screens. The onboarding flow pulls several helpers from `codex-core`:
   - `git_info::get_git_repo_root` and `resolve_root_git_project_for_trust` to scope the
     trust decision.
   - `config::set_project_trusted` to persist the choice.
   - `auth::{login_with_api_key, read_openai_api_key_from_env}` to facilitate sign-in.
7. **Resume selection** – Using `RolloutRecorder::list_conversations` and
   `find_conversation_path_by_id_str`, `run_main` determines whether to start fresh, resume
   the most recent session, or present the resume picker.
8. **Session log** – `session_log::maybe_init` consults the config to decide whether to
   record a JSONL log; it writes the files under the same log directory returned by core.
9. **Model upgrade prompt** – When the GPT-5 rollout banner should appear, the TUI updates
   `Config`, persists the selection via `persist_model_selection`, and flips the flag in
   `InternalStorage`.
10. **App startup** – Finally the function constructs the `App` and enters the async event
    loop in `App::run`.

## 3. Conversation lifecycle wiring (`codex-rs/tui/src/app.rs`)
- `App::run` creates an `AppEvent` unbounded channel, a `ConversationManager`, and the
  initial `ChatWidget`.
- For fresh sessions, `ChatWidget::new` uses `chatwidget::agent::spawn_agent`, which:
  1. Calls `ConversationManager::new_conversation(config)`.
  2. Immediately forwards the captured `SessionConfiguredEvent` into the app as an
     `AppEvent::CodexEvent` so the UI can render the resume banner before any other event.
  3. Spawns two async tasks: one pulls `Op`s from the UI channel and passes them to
     `CodexConversation::submit`, while the other drives
     `CodexConversation::next_event` and forwards every `Event` as an `AppEvent::CodexEvent`.
- When resuming from rollout data, `ConversationManager::resume_conversation_from_rollout`
  reconstructs the server-side history before the same loops are spawned by
  `spawn_agent_from_existing`.
- The main select loop in `App::run` consumes:
  - `AppEvent`s (driven by core events, file-search tasks, approval widgets, etc.).
  - Terminal events (`TuiEvent`) such as key presses or draw ticks. The TUI always keeps
    the `ConversationManager` behind an `Arc`, so multiple tasks and widgets can share it.

## 4. Outbound operations (TUI → core)
Every outgoing `Op` is funneled through `ChatWidget::submit_op`, which also records the
payload in the session log. The most important producers are:

| Producer | Location | Purpose | `Op` variants |
|----------|----------|---------|---------------|
| Chat composer | `chatwidget.rs` | Submit user messages, request slash commands, or compact history | `UserInput`, `AddToHistory`, `ListCustomPrompts`, `Compact`, `ListMcpTools`, `Interrupt`, `Shutdown` |
| Model selector popup | `chatwidget.rs` | Switch model or reasoning effort mid-session | `OverrideTurnContext` (with `model`/`effort` fields) |
| Approval-mode popup | `chatwidget.rs` | Toggle approvals/sandbox presets | `OverrideTurnContext` (with approval & sandbox overrides) |
| Status indicator | `status_indicator_widget.rs` | Allow Esc to forward interrupts while a task runs | `Interrupt` |
| User approval modal | `user_approval_widget.rs` | Respond to core-sent approval requests | `ExecApproval`, `PatchApproval` |
| Chat history navigator | `bottom_pane/chat_composer_history.rs` | Fetch persisted message history on demand | `GetHistoryEntryRequest` |
| Backtrack controller | `app_backtrack.rs` | Request the full conversation path to fork | `GetPath` |
| Session log | `session_log.rs` | Records the absolute workspace path backing the current session | `GetPath` (mirrors the backtrack request) |

Because `codex_core::protocol::Op` is re-exported from `codex-core`, these sites do not
need to depend on `codex-protocol` directly.

## 5. Inbound events (core → TUI)
`ChatWidget::handle_codex_event` owns the message demultiplexer. Key responsibilities:

- **Session setup** – `SessionConfiguredEvent` seeds model metadata, shared history IDs,
  the welcome banner, and triggers a `ListCustomPrompts` request.
- **Streaming content** – `AgentMessageDelta`, `AgentReasoningDelta`,
  `AgentReasoningRawContentDelta`, and `ExecCommandOutputDelta` pipe through the streaming
  controller so partial content renders smoothly.
- **Plan updates** – `PlanUpdate` events append structured plan information to the
  transcript using `history_cell::new_plan_update`.
- **Command execution lifecycle** – `ExecCommandBegin`, `ExecCommandEnd`,
  `ExecApprovalRequest`, `ExecCommandOutputDelta`, and `TurnDiff` events drive the inline
  history cells, approval widgets, and status indicator.
- **Patch workflow** – `ApplyPatchApprovalRequest`, `PatchApplyBegin`, and
  `PatchApplyEnd` surface inline diff previews, queue approval UIs, and record outcomes.
- **Tooling** – `McpToolCallBegin/End`, `McpListToolsResponse`, `ListCustomPromptsResponse`
  populate auxiliary widgets such as the slash-command popup.
- **Environment context** – `ConversationPath` events are forwarded to
  `AppEvent::ConversationHistory`, enabling the backtrack overlay to render accurate
  previews.
- **Errors & interrupts** – `Error`, `StreamError`, and `TurnAborted` map to history
  entries and composer state updates so the user can retry.
- **Token accounting** – `TokenCount` updates the composer placeholder and status footer.

Any event that must alter global UI state (for example model updates) sends a secondary
`AppEvent` back into the main loop so shared state like `App::config` stays synchronized.

## 6. Session history, backtracking, and resume
- **Rollout recorder** – `codex_core::RolloutRecorder` writes every session to disk. The
  resume picker (`resume_picker.rs`) uses `RolloutRecorder::list_conversations` and
  displays metadata pulled from the rendered rollout.
- **Session replay** – When resuming, `ChatWidget::new_from_existing` replays the initial
  messages contained in `SessionConfiguredEvent.initial_messages`. These replays are marked
  with a synthetic `Event` ID so downstream handlers can skip side effects.
- **Composer history** – `ChatComposerHistory` mixes locally submitted prompts with
  persisted history retrieved via `Op::GetHistoryEntryRequest` and reacts to
  `GetHistoryEntryResponse` events to populate the composer.
- **Backtrack flow** – `AppBacktrack` tracks Esc presses, issues `Op::GetPath` when the
  user confirms, and waits for a `ConversationPathResponseEvent`. Once received, it calls
  `ConversationManager::fork_conversation` with the historic rollout path.

## 7. Approvals, sandboxing, and configuration persistence
- The approval modal and status indicator surface `codex_core::protocol::AskForApproval`
  and `SandboxPolicy` defaults taken from the runtime `Config`.
- Selecting a different preset triggers both a core-side override (`Op::OverrideTurnContext`)
  and local state updates (`AppEvent::UpdateAskForApprovalPolicy`,
  `AppEvent::UpdateSandboxPolicy`).
- Model changes follow the same pattern and persist to disk via
  `codex_core::config::persist_model_selection` scoped to the active profile.
- Trust decisions in the onboarding flow call `config::set_project_trusted`, which writes
  `trusted_projects` entries into the user’s config directory.

## 8. Shared helpers relied upon by the TUI
- **Authentication** – `AuthManager` abstracts chatgpt/api-key storage and refresh. The
  onboarding UI uses it to kick off OAuth or copy API keys.
- **Git helpers** – `git_info::{get_git_repo_root, resolve_root_git_project_for_trust}`
  ensure trust settings apply to the correct repository root.
- **Token data** – `CodexAuth` / `token_data::TokenData` provide structured access to
  stored id/access tokens when determining login status.
- **Environment context** – `config::Config` exposes `cwd`, `codex_home`, sandbox
  settings, and `model_provider` metadata that the UI reflects in headers and tooltips.

## 9. Logging and diagnostics
- `session_log` records a rich JSONL log when enabled. Each `AppEvent` and outbound `Op`
  is serialized so that a session can be replayed offline. The log lives next to the main
  `codex-core` log directory for discoverability.
- Panic handling in `run_ratatui_app` replaces the default hook so that panic summaries are
  surfaced inside the TUI before falling back to `color-eyre`’s detailed report.
- `updates` (release-only) fetches version info, but the decision to print the banner stays
  inside the TUI; core only provides the configuration paths needed to read/write state.

## 10. Extending the integration safely
When adding new features that span both crates:
1. Prefer extending the existing `Op`/`Event` enums in `codex-protocol`; the TUI already
   assumes every piece of cross-crate communication flows through them.
2. Keep the TUI-side submission logic inside `ChatWidget::submit_op` so session logging and
   error reporting stay consistent.
3. If a new flow needs persisted settings, add them to `codex_core::config::Config` (and
   the TOML structs) so the TUI can adopt them without duplicating parsing logic.
4. Reuse `AppEvent` as the cross-cutting message bus inside the TUI. Widgets should send
   domain events to `AppEventSender` rather than talking directly to `ConversationManager`.
5. Make sure long-running background tasks that call into core keep the `AuthManager` and
   `Config` copies thread-safe (`Config` is `Clone` for this purpose).

---

The pairing of `codex-tui` and `codex-core` is intentionally thin: the UI obtains all of
its business logic from the core crate, and the core crate treats the UI as just another
client that understands the public protocol. Leaning on the existing hooks keeps that
separation intact while providing enough surface area for rich terminal features.
