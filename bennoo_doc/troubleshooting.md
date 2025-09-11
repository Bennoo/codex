# Troubleshooting & Issue Handling

When users report problems or you encounter failing tests, follow these steps to diagnose and resolve the issue.

## Reproducing Problems

1. **Gather context** – confirm the OS, CLI version, and steps to reproduce. Ask users for command output or logs.
2. **Isolate the environment** – run inside a clean container or using `--sandbox danger-full-access` only if necessary.
3. **Use verbose logging** – set `RUST_LOG=debug` and collect the output.

## Reading Logs

Logs use the [`tracing`](https://docs.rs/tracing) crate. Common levels:

- `error` – serious failures
- `warn` – unexpected but recoverable issues
- `info` – high-level progress
- `debug`/`trace` – detailed diagnostics

## Common Issues

### Authentication Failures

- Ensure the user has a valid ChatGPT session or API key.
- For browser-based login, check that the callback URL is whitelisted.

### Sandbox Errors

- Commands may be denied by the sandbox policy. Advise using a less restrictive policy (`--sandbox workspace-write` or `--sandbox danger-full-access`) only in trusted environments.
- For Linux landlock errors, ensure the `codex-linux-sandbox` binary is installed and executable.

### Patch Application Problems

- Patches are applied with the `codex-apply-patch` crate. Confirm that diff headers and paths are correct.
- Use `codex debug apply-patch` to test problematic diffs.

### TUI Rendering Glitches

- Snapshot tests in `codex-rs/tui/tests/` help reproduce rendering bugs. Run `cargo test -p codex-tui` and review `.snap` files.
- Terminal compatibility issues may stem from missing Unicode or color support.

## Debugging Strategies

- Add temporary `tracing::info!` calls to narrow down logic errors.
- Use `cargo run -p <crate> -- <args>` to execute binaries directly.
- Write small integration tests under `tests/` to capture regressions.

## Handling User Issues

1. Reproduce the problem locally.
2. Check open issues or discussions to avoid duplicates.
3. Provide clear reproduction steps and propose a fix or workaround.
4. Link relevant documentation (e.g., files under `docs/`).

## Reporting Security Concerns

If an issue might expose a security vulnerability, follow the repository’s security policy and contact `security@openai.com` rather than opening a public issue.

Being systematic and communicative helps maintainers resolve issues quickly and keeps the community healthy.
