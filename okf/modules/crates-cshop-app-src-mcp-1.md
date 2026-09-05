---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-app/src/mcp (base64, editor,
  http, json, mod, protocol, reference)'
files:
- crates/cshop-app/src/mcp/base64.rs
- crates/cshop-app/src/mcp/editor.rs
- crates/cshop-app/src/mcp/http.rs
- crates/cshop-app/src/mcp/json.rs
- crates/cshop-app/src/mcp/mod.rs
- crates/cshop-app/src/mcp/protocol.rs
- crates/cshop-app/src/mcp/reference.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-app/src/mcp (base64, editor, http, json, mod, protocol, reference)
type: Module
---

### What it does
Implements the Model Context Protocol (MCP) server that exposes C-Shop's image editing capabilities over HTTP/1.1 with JSON-RPC 2.0 messaging. Allows remote clients to drive the editor through a stateful session model, receiving rendered images back as base64-encoded PNGs.

### Public interface
**Editor thread management:**
- `Editor::start(workspace) -> Result<Editor, String>` — starts the GPU-backed editing thread; fails if no GPU available
- `Editor::submit(session, work, want_image) -> Outcome` — submits work to the editor and waits for completion

**HTTP server:**
- `http::serve(listener, handle)` — accept loop that spawns one thread per connection, calls `handle(&Request) -> Response`

**JSON-RPC protocol:**
- `protocol::parse(body) -> Incoming` — parses JSON-RPC 2.0 requests/notifications
- `protocol::result(id, value) -> String` — formats successful response
- `protocol::error(id, code, message) -> String` — formats error response
- `protocol::initialize(params) -> Json` — generates MCP initialization response

**JSON manipulation (hand-written):**
- `json::parse(source) -> Result<Json, String>` — recursive descent parser with depth limit
- `Json::get(key)`, `Json::as_str()`, `Json::as_f64()`, `Json::as_array()` — accessors
- `Json::write() -> String` — serializes to JSON text

**Base64 encoding:**
- `base64::encode(bytes) -> String` — standard alphabet with PKCS#7 padding

### Key invariants
- One GPU context and one editor thread per process; all script execution is serialized onto that thread to prevent document races.
- Sessions are keyed by caller-provided string and automatically expire after 30 minutes of inactivity; LRU eviction when the session limit (32) is reached.
- HTTP connections support keep-alive for up to 512 requests before being asked to reconnect, with timeouts on all socket operations.
- Images returned to callers are always base64-encoded PNGs within JSON; full-resolution composites never leave the server unscaled.
- JSON parser recursion is limited to 64 levels deep; chunked HTTP bodies are rejected (only Content-Length accepted).

### Non-obvious decisions
- **Hand-written JSON and HTTP libraries**: Chosen to keep the crate dependency-free. Protocol-facing parsers are written to make failure modes visible rather than hidden in transitive dependencies.
- **Blocking thread-per-connection model instead of async**: The GPU render work is already serialized onto one thread; async concurrency above that buys nothing and would introduce a large async runtime as a dependency.
- **Session expiry rather than unbounded session storage**: Holding GPU textures indefinitely for idle clients would leak memory. The 30-minute window is long enough for agent workflows with pauses between calls.
- **Scaling images before base64 encoding**: A full-resolution composite can be tens of megabytes; base64 expands by 33%, making the response prohibitively large. Scaling on the server keeps transfers small.
- **Objects preserve insertion order in JSON**: Callers (agents) read reports by eye; field shuffle on every serialization is confusing even if JSON semantics allow it.

### Unclear intent
None identified. All modules are referenced from `crate::script` (Runner, Sandbox) which is external to this module; tool dispatch is in `tools` submodule not shown here.
