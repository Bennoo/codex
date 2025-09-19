# codex-mcp-client

Client implementation for the Model Context Protocol (MCP) used by Codex to connect to MCP servers.

## Summary

- Provides async I/O, logging, and typed messaging around `mcp-types`.

## Depends On

- `mcp-types`, `serde(_json)`, `tokio`, `tracing`

## Used By

- `codex-core`

```mermaid
graph LR
  core[codex-core] --> mcpcli[codex-mcp-client]
  mcpcli --> mcpt[mcp-types]
```

