# codex-exec

Headless, non‑interactive executor for Codex workflows. Useful in CI/automation.

## Summary

- Runs Codex to completion from a prompt or stdin, printing results directly.
- Shares core logic with the TUI via `codex-core` and `codex-common`.

## Targets

- Bin: `codex-exec`
- Lib: `codex_exec`

## Depends On (internal)

- `codex-arg0`, `codex-common` (cli, elapsed, sandbox_summary), `codex-core`, `codex-ollama`, `codex-protocol`

## Used By

- Invoked via `codex exec` from `codex-cli` or directly.

```mermaid
graph LR
  cli[codex-cli] --> exec[codex-exec]
  exec --> core[codex-core]
  exec --> common[codex-common]
  exec --> arg0[codex-arg0]
  exec --> proto[codex-protocol]
  exec --> ollama[codex-ollama]
```

