---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-app/tests'
files:
- crates/cshop-app/tests/mcp.rs
- crates/cshop-app/tests/script.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-app/tests
type: Module
---

### What it does

Tests the scripting system and MCP server integration, covering JSON parsing, protocol negotiation, socket communication, sandboxing, and the full script-to-image pipeline including styles, effects, and document manipulation.

### Public interface

**mcp.rs:**
- `mcp::json::parse(source: &str) -> Result<Json, String>` — parses JSON with depth limits
- `mcp::json::Json::write() -> String` — serializes JSON to text
- `mcp::protocol::parse(input: &str) -> Incoming` — parses JSON-RPC messages
- `mcp::protocol::initialize(params: &Json) -> Json` — negotiates protocol version
- `mcp::tools::list() -> Json` — returns available tools with schemas
- `mcp::tools::TOOLS` — array of tool implementations
- `mcp::server::serve(config: Config) -> Result<(), String>` — runs HTTP server
- `mcp::server::Config` — server configuration (addr, workspace, token, allow_origins)
- `mcp::reference::describe(topic: &str) -> String` — returns reference documentation

**script.rs:**
- `script::run(source: &str, base: &Path) -> Result<Report, String>` — executes script and returns document + metadata
- `script::parse(source: &str) -> Vec<Result<Command, Error>>` — parses script into commands
- `script::parse_color(text: &str) -> Result<Rgba8, String>` — parses color from #hex or name
- `script::parse_style(source: &str) -> Style` — parses style file into parameters and body
- `script::substitute(text: &str, values: &[(String, String)]) -> Result<String, String>` — replaces {holes} with values, evaluates arithmetic
- `script::resolve(base: &Path, path: &str) -> PathBuf` — resolves paths relative to base, expands ~
- `script::Report` — contains ok flag, document dimensions, layers, steps, outputs, facts

### Key invariants

- A session ID persists a document across multiple tool calls; different sessions are isolated.
- Scripts cannot read, write, or escape their assigned workspace directory.
- Bearer token authentication is required when configured; the `/health` endpoint remains open.
- Cross-origin requests from non-localhost are rejected unless explicitly allowed.
- Non-loopback binding without a token is refused at startup.
- One failed command does not stop script execution; all steps are reported with individual success/failure status.
- A style's parameters must be declared before use; unknown parameters are rejected with a helpful error listing available ones.
- A style cannot recursively apply itself; depth is limited.
- Parameter overrides beat default values in styles.
- JSON output is always well-formed even when containing quotes, newlines, or backslashes in user data.
- Base64-encoded image data in responses is valid and decodable.

### Non-obvious decisions

- **Bare flags in positional arguments**: The parser does not distinguish flags from positional args unless they contain `=`, so `text 10 20 bold` puts `bold` in args rather than flagging it. This is deliberately permissive to avoid ambiguity in the grammar, with the rule that flags come last.

- **Arithmetic in substitution**: Parameter holes like `{n+n*2}` are evaluated as expressions rather than treated as plain text. This is non-obvious because it means a parameter value intended to be a blend mode name or bare word still works, but a parameter containing an expression is dangerous—yet the tests show this is intentional for scaling styles to documents.

- **Whole number options accept decimals from arithmetic**: An option typed as u32 accepts `3.0465` and truncates it. This is non-obvious because it trades strictness for composability: a style scaling to image dimensions via arithmetic must be able to pass decimal results to integer parameters.

- **Notifications return 202 with empty body**: JSON-RPC notifications (no id field) are answered with HTTP 202 Accepted and no JSON body, rather than 200 OK with a response. This signals that the call was accepted but no reply is owed, which is a protocol-level signal invisible in the JSON itself.

- **Session ID is returned in a response header, not JSON**: The session identifier travels back on initialize via the `Mcp-Session-Id` HTTP header rather than in the JSON-RPC result. This allows stateless load balancers to route follow-up calls to the same server without parsing the response body.

- **Styles can reference document dimensions as {width} and {height}**: A style parameter can use the bound document's size even though it was never explicitly passed. This requires styles to be evaluated in a context that includes document state, which is non-obvious because parameters are normally static.

### Unclear intent

- The `allow_origins` field in `Config` is always empty in tests but the code path for checking it exists; unclear whether CORS allowlisting is partially implemented or reserved for future use.
- The test `a_colour_blending_keeps_the_backdrop's_li` is cut off mid-sentence in the source; its actual purpose is unresolvable from the provided text.
