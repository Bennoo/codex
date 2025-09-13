# Running the Codex CLI Locally

This guide explains how to run the Codex CLI from source as a developer. It walks through setting up dependencies, building the Rust workspace, and launching the interactive TUI.

## Prerequisites

- **Rust** 1.84 or newer. Install with [rustup](https://rustup.rs/).
- **Cargo**, included with rustup.
- **Node.js** and **pnpm** ≥ 9 (optional, used for formatting scripts).
- An **OpenAI account** with either a ChatGPT plan or an API key.

## Clone the repository

```bash
git clone https://github.com/openai/codex.git
cd codex
```

## Build the CLI

The Rust workspace lives in `codex-rs`.

```bash
cd codex-rs
cargo build -p codex-cli
```

After a successful build, the `codex` binary is located at `target/debug/codex`.

## Run the CLI

Start an interactive session using cargo:

```bash
cargo run -p codex-cli
```

The first launch prompts you to authenticate. Choose one of:

- **Sign in with ChatGPT** – follow the browser flow.
- **API key** – set an environment variable and skip browser login:

  ```bash
  export OPENAI_API_KEY="your-key"
  cargo run -p codex-cli
  ```

  You can also run `cargo run -p codex-cli -- login --api-key "your-key"` to store credentials.

### Passing a prompt directly

Provide an initial prompt after a `--` to bypass the interactive composer:

```bash
cargo run -p codex-cli -- "explain this codebase to me"
```

### Choosing a working directory

By default Codex uses the current directory as the workspace. Override it with `--cd` (or `-C`):

```bash
cargo run -p codex-cli -- --cd path/to/project
```

### Debug logging

Enable verbose logs for troubleshooting:

```bash
RUST_LOG=debug cargo run -p codex-cli
```

## Configuration

Codex stores configuration in `~/.codex/config.toml`. Common settings include the model, provider, and sandbox policy. See [`docs/config.md`](../docs/config.md) for the full reference. Example:

```toml
model = "gpt-5"
sandbox_mode = "workspace-write"
```

## Running the built binary

Instead of `cargo run`, execute the compiled binary directly:

```bash
./target/debug/codex --help
```

## Helpful commands

- `cargo run -p codex-cli -- exec "task"` – run Codex in non‑interactive mode.
- `cargo run -p codex-cli -- completion bash` – generate shell completions (`bash`, `zsh`, `fish`).

For more troubleshooting tips, see [troubleshooting.md](./troubleshooting.md).
