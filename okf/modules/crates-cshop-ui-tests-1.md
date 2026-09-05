---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-ui/tests (clipboard, editing,
  effects, filters)'
files:
- crates/cshop-ui/tests/clipboard.rs
- crates/cshop-ui/tests/editing.rs
- crates/cshop-ui/tests/effects.rs
- crates/cshop-ui/tests/filters.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-ui/tests (clipboard, editing, effects, filters)
type: Module
---

### What it does

Integration tests for the cshop-ui editing layer, covering clipboard operations, document editing, layer effects, and image filters. These tests drive the application's command system and compositor directly without a window, using a headless GPU context where needed.

### Public interface

`Harness::new(size: (u32, u32)) -> Option<Harness>` — creates a test harness with an embedded CShopApp.

`Harness::settle(frames: usize)` — advances the compositor through n frames to complete rendering.

`Harness::press(key: &str)` — simulates a keyboard shortcut.

`Harness::click(pos: egui::Pos2)` — simulates a mouse click at screen coordinates.

`Harness::doc_to_screen(x: f32, y: f32) -> Option<egui::Pos2>` — converts canvas coordinates to screen space.

`CShopApp::dispatch(Action)` — executes a command (Copy, Paste, NewLayer, SetLayerEffects, etc.).

`CShopApp::begin_stroke`, `continue_stroke`, `end_stroke`, `cancel_stroke` — paint stroke lifecycle.

### Key invariants

- The clipboard is detached by default to prevent test interference; only one test touches the system clipboard.
- A document must always have at least one layer; DeleteLayer refuses to remove the last one.
- Changes to the same layer property within a history step merge into a single entry (e.g., slider drags).
- Undo/redo must restore the complete state, including layer counts, pixel data, and property values.
- Editing a layer with effects must expand the dirty region to cover the effect's reach (e.g., shadow spread).
- Effects are cleared when composing; fill opacity is applied during effect composition, not during final blending.
- A dialog being open blocks tool interaction on the canvas.
- Saving adopts the new file path and name, and clears the modified flag.

### Non-obvious decisions

Pixel editing directly (via `pixels_mut()`) requires explicit `mark_dirty()` and `invalidate()` calls because the command system cannot observe it automatically. This signals intent and prevents the composite from becoming stale.

Feathered selection copies soft edges by composing with the effect system, rather than storing separate alpha masks. This reuses the blur infrastructure.

Copy without an active selection falls back to the whole layer rather than failing, matching user expectations in similar applications.

Paste centers by default but PasteInPlace restores the original offset, providing both workflows without a dialog.

Effects enlarge `render_bounds()` but leave `bounds()` (the pixel bounds) unchanged, allowing the compositor to know the true drawing region without relocating the layer.

A document round-trip test exists for effects (`effects_survive_a_document_round_trip_through_the_compositor`) because the compositor is an external component (cshop-gpu) that could drop or corrupt styled layers.

### Unclear intent

The `ready()` helper function in effects.rs and filters.rs creates a white 200×200 canvas with a 40×40 gray square, but the rationale for these specific dimensions is not documented in the test code itself.
