---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-ui/src (panels)'
files:
- crates/cshop-ui/src/panels.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-ui/src (panels)
type: Module
---

### What it does

Implements the right-hand dock UI containing Layers, History, Color, Properties, and Channels panels. The Layers panel is the primary focus, featuring layer thumbnails, blend modes, opacity controls, visibility toggles, masks, locks, and drag-to-reorder functionality.

### Public interface

```rust
pub fn dock(app: &mut CShopApp, ui: &mut egui::Ui)
pub fn layer_row_id(id: LayerId) -> egui::Id
```

### Key invariants

- The Layers panel always occupies at least 180 pixels in height and at most (total - 120) pixels, with other panels sharing the space above it in a scrollable column.
- Layer rows are drawn top-first in the UI, but the tree stores layers bottom-first; insertion positions must account for this reversal.
- A layer drag is only valid if it does not move the layer to the same position, does not nest a layer into itself or its descendants, and does not place anything below a pinned Background layer.
- The active layer's edit target (Pixels or Mask) is only shown as selected when that layer is active; other layers show only the pixel plate outline.
- Disabled masks are struck through rather than hidden, so their existence remains visible.

### Non-obvious decisions

- Properties panel appears first in the scrolling area (above Color and History) when the active layer is an adjustment layer, because that is what the user just clicked on and what they want to edit.
- The insertion line for layer drag-to-reorder is drawn at a gap position that depends on which half of a row the pointer is in (row.center().y vs row.max.y), making the drop position visually unambiguous.
- The drag state is stored in `ui.ctx().data()` as a temporary value keyed by `drag_id()`, rather than being resolved immediately in the row, because only the panel can see all rows at once and thus verify the drop is legal; rows that cleared drag state before drawing rows beneath them used to swallow the drop.
- Blend mode and opacity controls appear above the layer stack rather than inline with each row, because there is only one active layer to edit at a time.
- Lock toggles are greyed out rather than hidden when unavailable, so the UI layout remains stable and the user can see why a command is disabled.

### Unclear intent

- The `section_fill` function (used for the Layers panel header) differs from `section` (used for Properties, Color, History, Channels) by not being collapsible and taking different layout logic, but the reason for this separation is not evident from the code alone.
