---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-app/src (main, screenshot)'
files:
- crates/cshop-app/src/main.rs
- crates/cshop-app/src/screenshot.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-app/src (main, screenshot)
type: Module
---

### What it does

C-Shop's application entry point and offscreen rendering pipeline. `main.rs` parses command-line arguments to route execution between interactive window mode, headless scripting, MCP server mode, and screenshot capture. `screenshot.rs` renders single frames to PNG without a display server, exercising the full rendering stack (compositor, egui, GPU) and providing demo scene builders for UI documentation.

### Public interface

**main.rs:**
- `fn main()` — Entry point; routes to window, script, MCP server, or screenshot based on CLI args.
- `fn parse_addr(given: &str) -> Result<std::net::SocketAddr, String>` — Parses `--serve` argument as full address, bare port, or default.

**screenshot.rs:**
- `pub fn capture(out: &Path, size: (u32, u32), files: &[String], frames: u32, setup: impl Fn(&mut CShopApp), clicks: Vec<(f32, f32, egui::PointerButton)>, drag: Option<(f32, f32, f32, f32)>) -> Result<(), Box<dyn std::error::Error>>` — Render `frames` offscreen frames and write last to PNG; apply setup closure, click events, and drag motion.
- `pub fn build_demo(app: &mut CShopApp)` — Assemble layered document with blend modes, masks, groups, opacity.
- `pub fn build_selection_demo(app: &mut CShopApp)` — Document with active elliptical selection, masked band, and clipped layer.
- `pub fn build_adjustment_demo(app: &mut CShopApp)` — Photo-like gradient base with stacked adjustment layers (Levels, Vibrance, PhotoFilter, Curves).
- `pub fn build_tools_demo(app: &mut CShopApp)` — Radial gradient, bucket fills, clone stamp strokes, bucket-filled swatches.
- `pub fn build_text_demo(app: &mut CShopApp)` — Text layers at various sizes, alignments, styles, with one layer mid-edit.
- `pub fn build_shape_demo(app: &mut CShopApp)` — Rectangles, circles, polygons in solid, outlined, and hollow styles.
- `pub fn build_effects_demo(app: &mut CShopApp)` — (File truncated; not shown in source.)

### Key invariants

- Screenshot capture runs exactly `max(frames, 1, clicks.len() * 3 + 12)` frames to allow interface settlement, animation, and multi-step interactions (menu open → item select).
- Pointer events in screenshots are staged: position moves one frame before press, press and release are two frames apart, so egui's hit-testing (which uses prior frame geometry) succeeds.
- Drag events span 6 frames of lerp from press point to drag target, so the capture shows motion in progress rather than completion.
- All demo scenes use fixed dimensions (860–900×520–600) and are designed to fit in the default `--size 1600x980` screenshot.
- Time advances in 60 FPS steps during capture (frame / 60.0 seconds) to allow egui fade-ins (~80 ms) and animations to reach visible states.
- MCP server mode never returns from `mcp::server::serve()`; script mode exits with status 0 on success, 2 on failed step, 1 on error; screenshot and window modes return errors to `main`.

### Non-obvious decisions

- **Separate frame loop for screenshots**: Screenshot capture does not use a simple one-shot render; it runs 30+ frames minimum to let egui's fade-in animations complete and textures register. A single-frame render would show half-transparent UI elements.
- **Pointer staging logic**: Clicks move the pointer one frame *before* the press and hold for two frames. This compensates for egui's hit-testing delay (it uses the prior frame's widget tree) and ensures clicks land on interactive elements rather than nothing.
- **Drag lerp over 6 frames**: Drags are linearly interpolated across multiple frames rather than snapped to the endpoint. This ensures the drag motion is visible in the final capture frame, showing the interaction mid-flight rather than completed.
- **Frame count formula**: The frame count is `max(frames, 1, clicks.len() * 3 + 12)` rather than a simple `frames` parameter. This ensures enough settling time (12 frames) plus 3 frames per click to open menus and navigate into submenus, so complex interactions remain visible.
- **Separate MCP module**: MCP server logic is split into `mcp::server` submodule rather than inline in `main`, allowing the server to stay resident while keeping main.rs's dispatch logic clean.

### Unclear intent

- `build_effects_demo` is declared in comments but the source is truncated; its exact scene composition is not visible.
