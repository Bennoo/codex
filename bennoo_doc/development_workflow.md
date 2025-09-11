# Development Workflow

This guide explains how to build, test, and submit changes to the Codex repository.

## Tooling Overview

- **Rust** ≥ 1.84 is required. A `rust-toolchain.toml` pins the version when using `rustup`.
- **Cargo** – builds individual crates.
- **Just** – a command runner with recipes in `codex-rs/justfile` for formatting and linting.
- **pnpm / Node** – used for formatting markdown and JSON files.

## Setting Up

1. Install [Rustup](https://rustup.rs/) and ensure `cargo` is on your `PATH`.
2. Install [pnpm](https://pnpm.io/) ≥ 9 if you plan to run formatting scripts.
3. Clone the repository and enter it:
   ```bash
   git clone https://github.com/openai/codex.git
   cd codex
   ```

## Building the CLI

The Rust workspace lives in `codex-rs`. To build the primary binary:

```bash
cd codex-rs
cargo build -p codex-cli
```

This produces a `codex` executable under `target/debug/`.

## Formatting and Linting

### Rust

Whenever you edit Rust code:

```bash
cd codex-rs
just fmt                    # run rustfmt
just fix -p <crate>         # run clippy for the crate you touched
```

`just fix` runs `cargo clippy` with repo-specific lints (see `AGENTS.md`).

### Markdown & JSON

From the repo root:

```bash
pnpm format
```

This checks `*.json`, `*.md`, `.github/workflows/*.yml`, and `**/*.js`.  
For other files, run Prettier manually:

```bash
pnpm exec prettier --write path/to/file.md
```

## Running Tests

Run tests for the crate you changed:

```bash
cd codex-rs
cargo test -p codex-tui         # example
```

If you touch shared crates (`common`, `core`, `protocol`), run the full suite:

```bash
cargo test --all-features
```

Some tests are skipped automatically in the sandbox environment via `CODEX_SANDBOX` and `CODEX_SANDBOX_NETWORK_DISABLED` variables.

## Commit Guidelines

1. Keep commits focused and provide descriptive messages.
2. Ensure `git status` is clean before committing.
3. Do **not** amend or force-push to existing commits when collaborating; instead add new commits or use GitHub’s squash merge.

Example commit workflow:

```bash
git add path/to/file
git commit -m "docs: add rust fundamentals guide"
```

## Submitting a Pull Request

1. Push your branch to GitHub.
2. Open a PR describing the change and referencing relevant issues.
3. Run all checks locally before requesting review.
4. Sign the CLA by commenting `I have read the CLA Document and I hereby sign the CLA` if prompted.

## Local Testing of the CLI

```bash
cd codex-rs
cargo run -p codex-cli -- --help
```

Use `--cd` or `-C` to specify the working directory when launching the CLI.

## Troubleshooting

- Run with verbose logging: `RUST_LOG=debug cargo run -p codex-cli`.
- If sandbox prevents a command, use `--sandbox danger-full-access` cautiously or run in an isolated container.
- For snapshot test updates in `tui`:
  ```bash
  cargo insta pending-snapshots -p codex-tui
  cargo insta accept -p codex-tui   # after reviewing
  ```

Following this workflow ensures consistent, high-quality contributions.
