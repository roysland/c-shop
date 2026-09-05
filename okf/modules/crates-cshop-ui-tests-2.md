---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-ui/tests (input, large_canvas,
  layer_drag, paths, selections, shapes)'
files:
- crates/cshop-ui/tests/input.rs
- crates/cshop-ui/tests/large_canvas.rs
- crates/cshop-ui/tests/layer_drag.rs
- crates/cshop-ui/tests/paths.rs
- crates/cshop-ui/tests/selections.rs
- crates/cshop-ui/tests/shapes.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-ui/tests (input, large_canvas, layer_drag, paths, selections,
  shapes)
type: Module
---

### What it does

Comprehensive integration tests for the C-Shop UI layer, covering user input routing, layer management, shape operations, selections, and performance regressions. These tests drive the real interface as a user would—clicking, dragging, and using keyboard shortcuts—to verify that widgets don't occlude each other, that painting respects selections, and that performance doesn't degrade with canvas size.

### Public interface

- `Harness::new(size: (u32, u32)) -> Option<Harness>` — Creates a test harness with a real UI context and GPU rendering.
- `Harness::click(pos: (f32, f32))` — Simulates a left mouse click at screen coordinates.
- `Harness::secondary_click(pos: (f32, f32))` — Simulates a right mouse click.
- `Harness::drag(from: (f32, f32), to: (f32, f32), steps: usize)` — Simulates a mouse drag across multiple frames.
- `Harness::press(chord: cshop_ui::shortcuts::Chord)` — Simulates a keyboard press.
- `Harness::settle(frames: usize)` — Runs the UI for N frames without input.
- `Harness::doc_to_screen(x: f32, y: f32) -> Option<(f32, f32)>` — Converts document coordinates to screen coordinates.
- `Harness::active_pixel(x: u32, y: u32) -> Option<Rgba8>` — Reads the pixel color at a document position in the active layer.
- `Harness::widget_center(id: egui::Id) -> Option<(f32, f32)>` — Returns screen coordinates of a widget's center.
- `CShopApp::open_document(doc: Document)` — Opens a new document in the application.
- `CShopApp::dispatch(action: Action)` — Dispatches a command action.
- `CShopApp::begin_stroke`, `continue_stroke`, `end_stroke` — Direct painting API for testing.
- `CShopApp::magic_wand_at(pos: Vec2, mode: SelectionMode)` — Triggers the magic wand selection tool.

### Key invariants

- Input always routes to the topmost interactive widget; widgets drawn last win click priority (prevents the regression where the title bar's own menus were dead).
- A brush stroke costs the same regardless of canvas size; layer snapshots, thumbnails, and selection bounds are computed only over the affected region, not the entire document.
- Selections confine all paint operations (stroke, fill, clear); unpainted pixels remain untouched.
- Layer reordering via drag-and-drop in the layers panel is a single undo step and respects the pinned Background layer.
- Boolean shape operations (union, subtract, intersect, xor) merge operands into one path layer with the chosen operation recorded.
- Quick Mask and selections are inverses: painting black in Quick Mask removes coverage; undoing restores the prior selection state.

### Non-obvious decisions

- **Harness settles for multiple frames after document open and after certain operations** — UI initialization, GPU texture uploads, and history label recording are asynchronous. Three frames is a stable baseline that allows these side effects to complete without being flaky on loaded machines.
- **`stroke_cost` takes the best of three runs rather than averaging or taking one** — The first run includes warm-up overhead; the shortest run has the least interference from background OS activity. This isolates the algorithm's cost rather than machine load.
- **Thumbnail scaling tolerance is 12x rather than a smaller ratio** — The performance regressions this guards against were 50x–150x; 12x is generous enough that a temporarily loaded machine won't fail CI while still catching real algorithmic regressions.
- **Magic wand and selection boolean modes don't panic on empty documents** — Rather than requiring a document to be open, actions gracefully become no-ops. This matches user expectations (pressing Select All does nothing visible if there's nowhere to select) and simplifies command dispatch.

### Unclear intent

None identified. All test purposes are clearly stated in module and test docstrings. Symbol origins from other crates (`cshop_core`, `cshop_ui`, `cshop_gpu`) are well-documented in the module listing.
