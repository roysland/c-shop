---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-ui/src (canvas)'
files:
- crates/cshop-ui/src/canvas.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-ui/src (canvas)
type: Module
---

### What it does

Renders the document viewport with a composited image, checkerboard background, and all pointer interactions. Handles canvas navigation (pan/zoom), tool-specific interactions for painting, selections, transforms, text, and shape tools, plus visual overlays for selections, transforms, and crop boxes.

### Public interface

```rust
pub fn show(app: &mut CShopApp, ui: &mut egui::Ui)
```

### Key invariants

- The canvas viewport must be computed before any tool interaction or rendering occurs, to ensure pointer coordinates map correctly to document space.
- A live transform or crop takes absolute precedence over all tool interactions; these are checked before tool-specific handlers.
- Modifiers (Shift, Alt, Ctrl) captured at drag start determine selection boolean mode; during the drag they constrain the shape instead.
- The checkerboard pattern is aligned to screen space (not document space) so it does not shimmer during canvas panning.
- Selection outlines are cached on the selection object and only retraced when the selection changes; the marching-ants animation is driven by a repaint request.
- Space-key panning and middle-mouse panning override any active tool and must be checked before tool dispatch.
- A non-modal dialog (Layer Style) can be open; when it is, canvas clicks are rejected to prevent stray strokes.
- Pointer position must be sampled inside the response's interact window; releasing outside the canvas still ends a stroke if painting.

### Non-obvious decisions

- **Tessellated transform mesh**: The transformed layer is subdivided into a 12×12 grid and drawn as triangles rather than a single textured quad. This is because affine interpolation on two triangles would visibly shear the image under a perspective transform; a grid of small triangles approximates the projective mapping closely enough.
- **Coarse quick-mask sampling**: Quick Mask is drawn by sampling the selection every 6 document pixels (or 1 pixel at high zoom) and rendering blocks, rather than uploading a full texture each stroke. This trades pixel-perfect accuracy for the practical observation that quick mask is a transient editing aid.
- **Dash phase animation via repaint requests**: The marching-ants outline does not animate continuously; instead it requests a repaint only when a selection or in-progress drag exists. When idle, no animation occurs and the editor does not waste CPU.
- **Modifier capture at selection drag start**: The boolean mode (replace/add/subtract/intersect) is determined from modifiers at the moment the drag starts, not continuously. During the drag, modifiers control shape constraints (square/circle, from-center) instead. This separation is intentional: it prevents accidentally changing selection mode mid-gesture.
- **Clone stamp anchor via Alt-click before drag**: Setting a clone source is detected at `drag_started` or `clicked`, not during the drag. This ensures the anchor is fixed before any pixels are sampled, preventing the source from shifting mid-stroke.
- **Whole-pixel-only Move tool offsets**: The Move tool rounds drag deltas to whole pixels and rejects sub-pixel offsets. Sub-pixel layer offsets would require resampling, which is deferred to the transform tool.

### Unclear intent

- **`SelectionDrag::Marquee` start/current mutation during drag**: The `start` and `current` fields are rewritten in-place during a shift or alt constrained drag to store the resulting constrained rect's corners. It is unclear why the constraint is applied by mutating the drag state rather than computing the constrained shape once and storing it explicitly.
