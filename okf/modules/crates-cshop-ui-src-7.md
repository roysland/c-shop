---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-ui/src (filter_ui, icons)'
files:
- crates/cshop-ui/src/filter_ui.rs
- crates/cshop-ui/src/icons.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-ui/src (filter_ui, icons)
type: Module
---

### What it does

`filter_ui.rs` implements a dialog for adjusting filter parameters with a live preview that stays responsive by rendering only a viewport-sized window. `icons.rs` provides vector icons for toolbar tools, drawn procedurally to remain crisp at any scale and automatically adopt the interface's colour scheme.

### Public interface

**filter_ui.rs:**
- `FilterDialog::new(filter: Filter, source: PixelBuffer, context: FilterContext) -> Self` — Creates a dialog for a given filter and source region.
- `FilterDialog::title(&self) -> String` — Returns the filter's display name.
- `FilterDialog::ui(&mut self, ui: &mut egui::Ui, actions: &mut Vec<Action>) -> bool` — Renders the dialog UI; returns true when it should close.
- `FilterDialog::zoom_to(&mut self, zoom: f32)` — Sets zoom from external callers (e.g., screenshot flags).
- `filter_editor(ui: &mut egui::Ui, filter: &mut Filter) -> bool` — Renders parameter controls for any filter variant; returns true if changed.

**icons.rs:**
- `tool(painter: &Painter, rect: Rect, tool: Tool, color: Color32)` — Draws the icon for a given tool into the specified rectangle.

### Key invariants

- The preview never renders more than `VIEW_W × VIEW_H` pixels at the current zoom, keeping interaction responsive regardless of source size.
- For bounded-support filters (blurs, sharpen, median), the preview is cut from full resolution with a margin so edge pixels see real neighbours; at 100% zoom it exactly matches the applied result.
- Whole-image filters (distortions, Average) always render the entire source at fit scale, with zoom only magnifying; settings are scaled to produce identical results regardless of preview scale.
- The margin around a preview crop is capped at `MAX_MARGIN` rendered pixels to prevent huge blur radii from turning a constant-cost preview into a full-image render.
- Icon coordinates are authored in a 0..1 square and mapped onto the target rect with consistent inset, so one definition works at any size.

### Non-obvious decisions

- **Margin on previewed crops**: Rather than naively cropping to the view, the code inflates the crop by the filter's support radius (capped) so that pixels at the crop edge still see real neighbours. This is the only way to make a zoomed local-filter preview exactly match the full-resolution applied result, not an approximation.
- **Separate handling of bounded vs. whole-image filters**: Bounded filters crop and render at the current zoom; whole-image filters render the entire source at fit scale. This is necessary because distortions and certain global effects are anchored to the image extent, so cropping would change the result.
- **Texture reuse instead of allocation per frame**: `set_texture` updates an existing texture in place rather than allocating a fresh one each frame. This avoids stutter during panning, which runs on every input event.
- **Press-moved flag**: Panning and the hold-to-compare gesture both use the same mouse button, so the code tracks whether a press has moved to distinguish a stationary hold (compare) from a drag (pan).
- **Zoom follow in fit mode**: When `fit` is true, the zoom is recomputed each frame rather than cached, so resizing the source dynamically adjusts the preview fit.
- **Icon colour parameters**: Icons take a `color` argument rather than reading from a theme, allowing the same drawing code to work for active, inactive, and hover states.

### Unclear intent

- The `source_size` field in the egui `ColorImage` (set to match pixel dimensions) — its purpose in egui is undocumented in this context, though it likely affects texture coordinate scaling for high-DPI displays.
