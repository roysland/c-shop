---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-ui/tests (transforms)'
files:
- crates/cshop-ui/tests/transforms.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-ui/tests (transforms)
type: Module
---

### What it does

End-to-end integration tests for the transform, crop, resize, and adjustment features of the C-Shop UI. Tests verify that layer transformations (rotation, flipping, free transform), canvas operations (crop, resize), and non-destructive adjustments work correctly through the application dispatch system and properly integrate with undo/redo.

### Public interface

```rust
fn app_with(w: u32, h: u32, bg: Background) -> Option<CShopApp>
fn layer(app: &CShopApp) -> &Layer
```

All tests use `app.dispatch(Action::...)` to exercise the public action interface defined in `cshop_ui::commands`.

### Key invariants

- A layer undergoing free transform must be hidden; its transform preview stands in until `CommitTransform` or `CancelTransform` returns it to visibility.
- Committing an untouched transform (one with no actual drag operations) records no history entry.
- Locked layers refuse to enter transform mode entirely.
- Layer masks must remain registered with the layer they mask; if a layer moves, its linked mask must move with it.
- Lossless transforms (90° rotations, flips) must preserve pixel data exactly and return to the original after four 90° rotations or two flips.
- `CropToSelection` and destructive adjustments must respect active selections; pixels outside a selection remain untouched.
- Canvas resize does not resample pixels; image resize resamples all layers uniformly.
- Adjustment layers inherit the current selection as their mask; destructive adjustments apply only to selected regions.
- Retuning an adjustment layer (changing its parameters repeatedly) collapses into a single history entry.
- The anchor grid arithmetic correctly shifts content when canvas size changes.
- Crop rectangles are clamped to canvas bounds on commit.

### Non-obvious decisions

- `app_with()` silently returns `None` on GPU context creation failure (headless mode) rather than panicking, allowing tests to skip gracefully when GPU is unavailable.
- Adjustment layers are added *above* the active layer (inserted at `root()[1]` for the second layer), establishing a consistent predictable order.
- Adjustments with no configurable settings (e.g., `Invert`) apply immediately via `ShowAdjustmentDialog`, while those with settings open a modal dialog first; this prevents apparent no-ops when users navigate the Adjustments menu.
- The history entry for repeated adjustment tuning is named after the adjustment type ("Brightness/Contrast"), not a generic "adjust" label, so users know what was modified.

### Unclear intent

- The `layer()` helper function fetches via `view.doc.active.unwrap()` without documenting the expectation that a document always has an active layer at test time; this is reasonable but unmarked.
