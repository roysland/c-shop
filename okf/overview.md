# C-Shop: GPU-Accelerated Image Editor

## What this project is

C-Shop is a GPU-accelerated raster and vector image editor built in Rust, offering layer-based composition, non-destructive adjustments, and extensibility via a Model Context Protocol server. It runs as a native desktop application with an egui interface, headless scripting environment, or HTTP-accessible remote editing service. The system prioritizes memory efficiency (tiling compositor, region-confined operations) and rendering correctness (CPU/GPU parity for blend modes and filters).

## Architecture

C-Shop follows a layered architecture with clear data flow:

**Document layer** (`cshop-core`) holds immutable document state: a canvas with layer tree, pixel/vector/text content, selections, and adjustment parameters. A separate history module manages reversible commands that mutate the document.

**Rendering layer** (`cshop-gpu`) composites the layer tree into final images on the GPU, with tiling to keep VRAM bounded. A compositor caches layer textures and applies blend modes, masks, and adjustments using wgpu shaders. The CPU filters layer (`cshop-core/filters`) provides destructive pixel operations as fallback.

**Persistence layer** (`cshop-io`) serializes documents to C-Shop's native format and reads/writes PSD files, with bounds-checking on all binary reads.

**UI layer** (`cshop-ui`) presents the application: a canvas viewport, toolbox, panels (layers/history/color/properties), and modal dialogs. It drives commands through the document and re-renders via the GPU compositor whenever state changes.

**Application entry point** (`cshop-app`) routes between GUI (window + egui), headless scripting, MCP server, and screenshot modes. The MCP server exposes editing capabilities over HTTP with JSON-RPC 2.0 messaging and session-based state isolation.

Data flows: user actions → UI commands → document mutations → dirty-state marking → compositor re-renders → display. Undo/redo mutate the document in reverse.

## Module map

- [./modules/crates-cshop-app-src-1.md](./modules/crates-cshop-app-src-1.md) → Application entry point routing between GUI, headless, MCP server, and screenshot modes
- [./modules/crates-cshop-app-src-3.md](./modules/crates-cshop-app-src-3.md) → Window management, GPU surface, and event loop coordination with winit, egui, and wgpu
- [./modules/crates-cshop-app-src-mcp-1.md](./modules/crates-cshop-app-src-mcp-1.md) → Model Context Protocol server with HTTP/1.1 transport and JSON-RPC 2.0 messaging
- [./modules/crates-cshop-app-src-mcp-2.md](./modules/crates-cshop-app-src-mcp-2.md) → MCP server session management, workspace isolation, and token authentication
- [./modules/crates-cshop-app-tests.md](./modules/crates-cshop-app-tests.md) → Integration tests for scripting and MCP server with socket communication and sandboxing
- [./modules/crates-cshop-core-examples.md](./modules/crates-cshop-core-examples.md) → Standalone examples and benchmarks for graphics operations
- [./modules/crates-cshop-core-src-1.md](./modules/crates-cshop-core-src-1.md) → Unified colour adjustment system with lookup tables and shader formulas for CPU/GPU parity
- [./modules/crates-cshop-core-src-10.md](./modules/crates-cshop-core-src-10.md) → 8-bit coverage masks for pixel selections with construction, boolean ops, and marching-ants outlining
- [./modules/crates-cshop-core-src-11.md](./modules/crates-cshop-core-src-11.md) → Vector shape rendering via SDFs, text layout with wrapping/alignment, and tile-by-tile snapshot capture
- [./modules/crates-cshop-core-src-12.md](./modules/crates-cshop-core-src-12.md) → Projective transformation matrices, hierarchical layer tree, and colour-based selection tools
- [./modules/crates-cshop-core-src-2.md](./modules/crates-cshop-core-src-2.md) → 27 W3C/PSD-compatible blend modes, sRGB/linear-light colour conversions, and monotone cubic tone curves
- [./modules/crates-cshop-core-src-3.md](./modules/crates-cshop-core-src-3.md) → Document model with canvas metadata, layer tree, selection tracking, and dirty-state marking
- [./modules/crates-cshop-core-src-6.md](./modules/crates-cshop-core-src-6.md) → Reversible command stack with memory budget and specialized variants for pixels, transforms, and metadata
- [./modules/crates-cshop-core-src-8.md](./modules/crates-cshop-core-src-8.md) → Brush stroke accumulation with flow/opacity separation and cubic Bézier path operations
- [./modules/crates-cshop-core-src-9.md](./modules/crates-cshop-core-src-9.md) → Tight-packed RGBA8 pixel buffers with region operations and multi-tap resampling filters
- [./modules/crates-cshop-core-src-filters-1.md](./modules/crates-cshop-core-src-filters-1.md) → Blur, distortion, and general effects (30+ filters) operating on CPU with parallel row processing
- [./modules/crates-cshop-core-src-filters-2.md](./modules/crates-cshop-core-src-filters-2.md) → Spatial filter infrastructure with preview-at-scale for interactive use
- [./modules/crates-cshop-core-src-filters-3.md](./modules/crates-cshop-core-src-filters-3.md) → Procedural image generation (clouds, fibers) using Perlin-like noise
- [./modules/crates-cshop-core-tests.md](./modules/crates-cshop-core-tests.md) → Integration tests comparing optimized implementations against references and validating history/selection bounds
- [./modules/crates-cshop-gpu-src-1.md](./modules/crates-cshop-gpu-src-1.md) → GPU device init, layer tree compositing, texture caching, and ping-pong tiling
- [./modules/crates-cshop-gpu-src-2.md](./modules/crates-cshop-gpu-src-2.md) → wgpu texture wrappers with format constants and convenience constructors
- [./modules/crates-cshop-gpu-tests.md](./modules/crates-cshop-gpu-tests.md) → GPU compositor validation against CPU references for blending, masking, and adjustments
- [./modules/crates-cshop-io-src-1.md](./modules/crates-cshop-io-src-1.md) → Binary serialization, format detection, and image encoding/decoding with bounds-checking
- [./modules/crates-cshop-io-src-2.md](./modules/crates-cshop-io-src-2.md) → Native `.cshop` project format serialization with forward-compatible tagged chunks
- [./modules/crates-cshop-io-src-3.md](./modules/crates-cshop-io-src-3.md) → PSD file reading/writing with layer groups, blend modes, masks, and PackBits compression
- [./modules/crates-cshop-io-tests.md](./modules/crates-cshop-io-tests.md) → Round-trip fidelity tests for `.cshop` and PSD formats
- [./modules/crates-cshop-ui-src-1.md](./modules/crates-cshop-ui-src-1.md) → Modal adjustment dialog with live preview on downscaled proxy region
- [./modules/crates-cshop-ui-src-2.md](./modules/crates-cshop-ui-src-2.md) → Main application state, frame update loop, document/tool/stroke/dialog management
- [./modules/crates-cshop-ui-src-3.md](./modules/crates-cshop-ui-src-3.md) → Canvas viewport rendering, checkerboard, pan/zoom, tool interactions, and overlay drawing
- [./modules/crates-cshop-ui-src-4.md](./modules/crates-cshop-ui-src-4.md) → Application chrome: title bar, toolbox, tool options, menus, and window controls
- [./modules/crates-cshop-ui-src-5.md](./modules/crates-cshop-ui-src-5.md) → Clipboard operations, color picker, command routing, and context menus
- [./modules/crates-cshop-ui-src-6.md](./modules/crates-cshop-ui-src-6.md) → Modal input dialogs and per-document GPU state (composites, transforms, thumbnails)
- [./modules/crates-cshop-ui-src-7.md](./modules/crates-cshop-ui-src-7.md) → Filter parameter dialog with viewport-sized live preview and procedurally-drawn tool icons
- [./modules/crates-cshop-ui-src-8.md](./modules/crates-cshop-ui-src-8.md) → Headless input simulation for UI testing, layer effects dialog, and public API exports
- [./modules/crates-cshop-ui-src-9.md](./modules/crates-cshop-ui-src-9.md) → Right-hand dock panels (Layers, History, Color, Properties, Channels) with layer thumbnails and drag-to-reorder
- [./modules/crates-cshop-ui-src-11.md](./modules/crates-cshop-ui-src-11.md) → Tool definitions and Free Transform/Crop operations with projective matrix manipulation
- [./modules/crates-cshop-ui-tests-1.md](./modules/crates-cshop-ui-tests-1.md) → Integration tests for clipboard, document editing, effects, and filters without a window
- [./modules/crates-cshop-ui-tests-2.md](./modules/crates-cshop-ui-tests-2.md) → Integration tests for input routing, layers, shapes, selections, and performance regressions
- [./modules/crates-cshop-ui-tests-3.md](./modules/crates-cshop-ui-tests-3.md) → Integration tests for keyboard shortcuts, text tool, and painting tools
- [./modules/crates-cshop-ui-tests-4.md](./modules/crates-cshop-ui-tests-4.md) → End-to-end tests for transforms, crop, resize, and non-destructive adjustments
- [./modules/crates.md](./modules/crates.md) → Example programs for GPU performance measurement and capability detection

## Getting started

1. Clone the repository and navigate to the project root.
2. Install a Rust toolchain (cshop targets a recent stable Rust).
3. Install system dependencies: a GPU driver and development headers (Linux: `libxcb-dev`, `libssl-dev`, etc.; macOS/Windows: typically bundled).
4. Run `cargo build --release` to compile all crates.
5. Launch the GUI with `cargo run --release -p cshop-app -- --window`, or try `cargo run --release -p cshop-app -- --screenshot` to render a demo without a display server.
6. To run tests: `cargo test --workspace`.

## Key design decisions

**Memory efficiency via tiling:** The compositor never allocates VRAM for the full document; instead it renders to fixed-size tiles and composites them to the final region. This keeps VRAM bounded regardless of document resolution.

**Region-confined selections and operations:** Masks and paint operations store data only in bounding regions, not document-sized buffers, reducing both memory and cache misses.

**CPU/GPU shader parity:** Blend modes, adjustments, and filters are implemented identically on CPU and GPU (via lookup tables and shader formulas) so offline rendering matches interactive preview.

**Forward-compatible binary formats:** The native `.cshop` project format uses tagged chunks; readers skip unrecognized chunks rather than failing, enabling safe format evolution.

**Stateless command design:** The history system stores reversible commands that know how to apply and revert themselves, decoupling undo/redo from the document structure.

**Headless testing and scripting:** The UI layer and compositor have no required display server; tests and the MCP server use headless rendering to avoid flakiness and enable distributed editing.

**Session-based MCP server:** Remote clients each get an isolated workspace and session token, with workspace-scoped file operations to prevent cross-session interference.