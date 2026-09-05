---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-ui/src (input_harness, layer_style,
  lib)'
files:
- crates/cshop-ui/src/input_harness.rs
- crates/cshop-ui/src/layer_style.rs
- crates/cshop-ui/src/lib.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-ui/src (input_harness, layer_style, lib)
type: Module
---

### What it does

`input_harness.rs` provides headless input simulation for testing the UI by scripting pointer and keyboard events and running them through the real egui hit-testing pipeline. `layer_style.rs` implements the Layer Style dialog, a tabbed interface for configuring layer effects with live preview on the canvas. `lib.rs` is the public API for the UI crate, re-exporting the main app and action types while providing utilities for document rendering and byte formatting.

### Public interface

**input_harness.rs:**
- `pub enum Step` — one frame of interaction (move, press, release, idle, key input)
- `pub struct Harness` — headless test harness with methods:
  - `pub fn new(size: (u32, u32)) -> Option<Harness>` — create harness if GPU available
  - `pub fn run(&mut self, steps: &[Step])`
  - `pub fn drag(&mut self, from: (f32, f32), to: (f32, f32), steps: usize)`
  - `pub fn settle(&mut self, frames: usize)`
  - `pub fn secondary_click(&mut self, at: (f32, f32))`
  - `pub fn press(&mut self, chord: crate::shortcuts::Chord)`
  - `pub fn click(&mut self, at: (f32, f32))`
  - `pub fn widget_center(&self, id: egui::Id) -> Option<(f32, f32)>`
  - `pub fn active_pixel(&self, x: i32, y: i32) -> Option<Rgba8>`
  - `pub fn doc_to_screen(&self, x: f32, y: f32) -> Option<(f32, f32)>`
  - `pub fn new_layer_button(&self) -> (f32, f32)`
  - `pub ctx: egui::Context`
  - `pub app: CShopApp`
  - `pub window_commands: Vec<WindowCommand>`

**layer_style.rs:**
- `pub struct LayerStyleDialog` with methods:
  - `pub fn new(layer: LayerId, effects: LayerEffects, name: String) -> Self`
  - `pub fn title(&self) -> String`
  - `pub fn ui(&mut self, ui: &mut egui::Ui, actions: &mut Vec<Action>) -> bool` — returns true when dialog should close

**lib.rs:**
- `pub use app::CShopApp`
- `pub use commands::Action`
- `pub fn format_bytes(bytes: u64) -> String`
- `pub fn render_document(gpu: &GpuContext, compositor: &mut Compositor, doc: &Document) -> PixelBuffer`

### Key invariants

- **Harness**: The frame counter increments on every frame, providing deterministic timing; egui texture deltas must be consumed (applied or freed) each frame to avoid panics; `window_commands` are mirrored from the app, with `WindowCommand::Close` setting `app.quit = true`.
- **LayerStyleDialog**: Changes apply to the document immediately (live preview); `before` snapshot is taken at construction for Cancel to restore; `touched` flag tracks whether any mutation has occurred, so Cancel only pushes an undo if needed; effects stack in a fixed order (Stroke topmost, DropShadow bottommost).
- **Input scripting**: Modifiers and keys require separate frames because egui's `InputState::modifiers` reflects only the last `ModifiersChanged` event of the current frame; pointer hit-testing uses the previous frame's geometry, so settlement frames are necessary after moving the pointer.

### Non-obvious decisions

- **Harness geometry computation**: `new_layer_button()` derives button position from window size rather than hard-coding coordinates, so tests survive panel width changes without modification.
- **LayerStyleDialog effect initialization**: Toggling an effect off deletes its state entirely, while toggling on creates a visible-by-default effect rather than a no-op. This is inferred from the `set_on` function's use of `then(Default::default)` and the comment about switching on as a request to view the effect.
- **Harness settlement requirement for secondary click**: After moving the pointer, two settlement frames precede the right-click because egui's geometry lags by one frame; this ensures hit-testing uses the current pointer position.

### Unclear intent

- **LayerStyleDialog `page` field initialization**: The logic opens on "whatever is already on" to avoid hiding an existing style, but the fallback to `Page::DropShadow` when no effect is enabled suggests a UI design preference that is not explicitly documented.
