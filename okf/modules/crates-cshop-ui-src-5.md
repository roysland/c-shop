---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-ui/src (clipboard, color_picker,
  commands, context_menus)'
files:
- crates/cshop-ui/src/clipboard.rs
- crates/cshop-ui/src/color_picker.rs
- crates/cshop-ui/src/commands.rs
- crates/cshop-ui/src/context_menus.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-ui/src (clipboard, color_picker, commands, context_menus)
type: Module
---

### What it does

This module provides UI components and command handling for C-Shop's interface layer, including clipboard operations, color selection, user action routing, and context menus. It bridges user interactions in the GUI with document state mutations and tool-specific behaviors.

### Public interface

**clipboard.rs**
- `struct Clipping` — Pixel data with canvas origin coordinates
- `struct Clipboard` — Dual-clipboard manager (system + internal)
  - `Clipboard::detached() -> Clipboard` — Headless/test mode without system clipboard
  - `Clipboard::has_content(&self) -> bool` — Check if paste is available
  - `Clipboard::set(&mut self, pixels: PixelBuffer, origin: (i32, i32))` — Copy to both clipboards
  - `Clipboard::get(&mut self) -> Option<Clipping>` — Retrieve paste, preferring internal copy if unchanged
  - `fn extract(src: &PixelBuffer, src_origin: (i32, i32), rect: IRect, coverage: impl Fn(i32, i32) -> f32) -> PixelBuffer` — Lift region with feathering

**color_picker.rs**
- `struct ColorPickerState` — HSV + alpha state across frames
  - `ColorPickerState::from_color(c: Rgba8) -> Self`
  - `ColorPickerState::to_color(self) -> Rgba8`
- `fn color_picker(ui: &mut egui::Ui, state: &mut ColorPickerState, original: Rgba8, with_alpha: bool) -> bool` — Draw picker UI, return true if changed

**commands.rs**
- `enum Action` — Complete set of user-triggered operations (document, history, tools, layers, editing, selections, adjustments, transforms, filters, masks)
- `enum WindowCommand` — Window chrome operations (drag, resize, minimize, maximize, close)
- `enum ResizeEdge` — Eight cardinal/diagonal edges for resize drags
- `enum TransformPreset` — Fixed rotations/flips (90°, 180°, horizontal/vertical flip)
- `enum Anchor` — Canvas resize origin (3×3 grid of positions)
- `enum ModifySelection` — Selection modification operations (feather, expand, contract, border, smooth)
- `impl Anchor::weights(self) -> (f32, f32)` — Distribution of size change per axis
- `impl Anchor::shift(self, from: (u32, u32), to: (u32, u32)) -> (i32, i32)` — Layer offset for canvas resize
- `impl Action::fill_foreground/fill_background(preserve_transparency: bool) -> Action` — Convenience constructors

**context_menus.rs**
- `fn canvas_menu(app: &mut CShopApp, ui: &mut egui::Ui)` — Right-click menu on canvas (tool-specific settings + commands)
- `fn layer_menu(app: &mut CShopApp, ui: &mut egui::Ui, id: LayerId)` — Right-click menu on layer row (blending, visibility, masks, effects)
- Helper functions: `heading()`, `slider()`, and tool-specific menus (`brush_menu()`, `marquee_menu()`, `wand_menu()`, `crop_menu()`, etc.)

### Key invariants

- **Clipboard identity**: When an image copied by C-Shop is pasted back without intervening system clipboard writes, the internal copy (with origin) is returned instead of the system version (without origin).
- **HSV state persistence**: `ColorPickerState` preserves hue and saturation even when they become undefined (black has no hue, white has no saturation), so dragging into corners doesn't lose the user's intent.
- **System clipboard resilience**: Once system clipboard initialization fails, subsequent operations skip it without retrying; only the internal clipboard remains available.
- **Action queueing**: All state mutations flow through `Action` enum variants, never directly in UI closures, keeping egui borrow semantics separate from application state.
- **Menu context**: Layer context menus automatically select the clicked layer if it differs from the currently active layer before showing menu-triggered actions.

### Non-obvious decisions

**Clipboard dual-store with content validation** (`clipboard.rs`): The module maintains both a system clipboard (for other applications) and an internal one (with origin data), then uses pixel-level comparison to detect when the user's own copy is pasted back. This avoids losing position metadata when round-tripping through the system clipboard, which cannot express canvas coordinates. The alternative—always trusting the system clipboard—would force "Paste in Place" to guess or ask the user.

**ColorPickerState HSV over RGB** (`color_picker.rs`): State is stored in HSV rather than converting back to RGB on every frame. When saturation is zero (gray) or value is zero (black), hue becomes mathematically undefined; storing it anyway preserves the user's hue choice if they then increase saturation or value, which is the expected behavior in color pickers.

**Anchor weights as normalized fractions** (`commands.rs`): When a canvas is resized, `Anchor::weights()` returns the fraction (0.0–1.0) of size change that lands before content (top-left) vs. after (bottom-right), per axis. This cleanly separates the geometric rule from pixel-specific rounding, which happens in `Anchor::shift()`.

**Detached clipboard mode** (`clipboard.rs`): Headless and test environments get a `Clipboard::detached()` that never touches system clipboard APIs. Without this, tests running in parallel would interfere via X11 selections; with it, each test session is isolated.

### Unclear intent

None identified. All symbols' purposes are clear from naming, doc comments, and surrounding code patterns.
