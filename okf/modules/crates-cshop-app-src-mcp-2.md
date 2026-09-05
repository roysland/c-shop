---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-app/src/mcp (server, tools)'
files:
- crates/cshop-app/src/mcp/server.rs
- crates/cshop-app/src/mcp/tools.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-app/src/mcp (server, tools)
type: Module
---

### What it does

Serves a C-Shop editor over HTTP using the Model Context Protocol (MCP), allowing remote clients to manipulate layered images through a REST-like JSON-RPC interface. Enforces workspace isolation, token authentication, and origin validation to safely expose filesystem operations across the network.

### Public interface

**server.rs**
```rust
pub struct Config {
    pub addr: SocketAddr,
    pub workspace: PathBuf,
    pub token: Option<String>,
    pub allow_origins: Vec<String>,
}

impl Default for Config

pub fn serve(config: Config) -> Result<(), String>
```

**tools.rs**
```rust
pub struct Tool {
    pub name: &'static str,
    pub title: &'static str,
    pub description: &'static str,
    pub schema: fn() -> Json,
}

pub const TOOLS: &[Tool]  // Six tools: run_script, render, list_styles, describe, workspace, reset

pub struct ToolResult {
    pub text: String,
    pub image_png: Option<Vec<u8>>,
    pub is_error: bool,
}

impl ToolResult {
    pub fn to_content(&self) -> Json
}

pub fn call(
    editor: &Editor,
    workspace: &Sandbox,
    name: &str,
    arguments: &Json,
    default_session: &str,
) -> ToolResult

pub fn list() -> Json  // The tools/list MCP response
```

### Key invariants

- **Workspace confinement**: Every path resolution goes through `Sandbox`, preventing directory traversal out of the workspace root.
- **Session isolation by default**: Requests without an explicit session ID get a unique, cryptographically-seeded session ID that persists only for that connection.
- **No remote origin without auth**: A non-loopback bind address requires a token; the server refuses to start otherwise.
- **Origin validation for browsers**: Requests carrying an `Origin` header are checked against loopback addresses or an explicit allowlist; requests with no `Origin` are permitted.
- **Token comparison in constant time**: Bearer token validation does not leak timing information about correct vs. incorrect prefixes.
- **One RPC answer per request**: The server does not offer server-initiated SSE streams; every tool call gets a synchronous response with optional image data.

### Non-obvious decisions

- **Session IDs are not credentials**: Session IDs use cryptographic mixing (splitmix64) to be unpredictable, but the actual security boundary is the bearer token. This allows session IDs to be safe identifiers without requiring them to be secrets.
- **Image size capped at 2048px on the wire**: Full-size renders go through `export` (a script command that writes to the workspace), not through tool results. This is because images in JSON-RPC are base64-encoded (33% overhead) and sending multi-megabyte images over JSON is impractical; the tool is for feedback, not delivery.
- **Six tools instead of one-tool-per-command**: `run_script` is the full editor, while the other five (`render`, `describe`, `list_styles`, `workspace`, `reset`) exist to let a cold client bootstrap itself without already knowing the domain. Wrapping individual commands as tools would duplicate the script language's own interface.
- **Token defaults to `None`, not a generated secret**: A loopback-only server is the safe default. Asking for a token when listening on 0.0.0.0 is deliberate friction to prevent accidental exposure.

### Unclear intent

None identified. The purpose of every function, check, and field is evident from naming, comments, or context.
