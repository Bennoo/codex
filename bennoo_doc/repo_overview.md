# Repository Overview

The Codex repository is a monorepo that hosts several tools for running OpenAI's Codex agent locally.  
It contains both Rust and TypeScript implementations, as well as extensive documentation and tooling.

```
├── codex-rs/         # Rust workspace with multiple crates
├── codex-cli/        # Legacy TypeScript CLI
├── docs/             # User-facing documentation
├── scripts/          # Maintenance scripts
├── AGENTS.md         # Instructions for contributors
├── package.json      # Node-based formatting tooling
└── ...
```

## Rust Workspace: `codex-rs`

`codex-rs` is a Cargo workspace containing the core implementation of the CLI and supporting libraries.
Below is a tour of the main crates:

### `ansi-escape`

Converts ANSI escape sequences into structures that can be rendered with [`ratatui`](https://ratatui.rs/). Used by the TUI to display colored output.

### `apply-patch`

Implements the `apply_patch` utility used by the agent to apply unified diff patches to the workspace. Exposes a library and a small binary for testing.

### `arg0`

Provides helpers for manipulating the program name (`argv[0]`). This allows the CLI to impersonate subcommands like `apply_patch` or sandbox wrappers.

### `chatgpt`

Contains code that interfaces with first‑party ChatGPT APIs. External contributors should coordinate with maintainers before modifying this crate.

### `cli`

The top‑level binary that end users invoke as `codex`. It wires together all other crates and exposes subcommands such as the TUI, `exec`, and the MCP server.

### `common`

Holds utilities shared across crates. Optional features enable CLI argument parsing, elapsed‑time reporting, and sandbox summaries.

### `core`

Business logic for Codex: task planning, tool invocation, sandbox awareness, and project documentation handling. Other crates depend heavily on `codex-core`.

### `exec`

A non‑interactive "headless" mode that runs a prompt to completion, suitable for scripting or CI usage.

### `execpolicy`

Evaluates sandbox policies written in [Starlark](https://github.com/bazelbuild/starlark) to restrict what commands Codex can run. Produces a binary and library.

### `file-search`

Provides fuzzy file searching used by the TUI via the `@` prompt shortcut.

### `linux-sandbox`

Implements Linux sandboxing (Landlock + seccomp). Exposes a CLI for debugging sandbox policies and a library used by `arg0` and other crates.

### `login`

Handles authentication flows for ChatGPT login and API key management. Includes a tiny HTTP server to receive callbacks.

### `mcp-client`

Model Context Protocol client implementation. Manages RPC‑style communication with external MCP servers.

### `mcp-server`

Allows Codex itself to act as an MCP server, exposing tools and information to other agents.

### `mcp-types`

Shared type definitions for the MCP protocol. Also generates TypeScript bindings via the `ts-rs` crate.

### `ollama`

Optional support for the [Ollama](https://ollama.com) local model runner. Streams responses and handles REST requests.

### `protocol`

Defines the Codex wire protocol and conversions between internal structures and MCP types. Uses `uuid`, `base64`, and `ts-rs` for cross‑language compatibility.

### `protocol-ts`

Command‑line tool and library for exporting protocol types to TypeScript. Ensures sync between the Rust and JS ecosystems.

### `tui`

The interactive terminal UI built with `ratatui`. Handles chat rendering, file views, diffing, and user input. Includes snapshot tests using `insta`.

## Legacy TypeScript CLI: `codex-cli`

The `codex-cli` folder hosts the older Node/TypeScript implementation. It remains for historical reference and packaging scripts but is superseded by the Rust CLI. Most new development happens in `codex-rs`.

## Documentation: `docs`

Markdown files that describe how to use Codex:

- installation guides
- configuration reference
- sandboxing and authentication docs
- advanced topics and FAQ

These documents are published as part of the open‑source project and are useful when responding to user questions.

## Other Top‑Level Files

- `PNPM.md`, `pnpm-lock.yaml`, and `pnpm-workspace.yaml` configure Node tooling used for formatting.
- `CHANGELOG.md` tracks notable updates.
- `cliff.toml` and `scripts/` provide release tooling.
- `AGENTS.md` defines repository‑wide contributor instructions that you are reading now.

Understanding this layout will help you find the right crate or script when fixing bugs or adding features.
