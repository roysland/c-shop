---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-core/src (paint, path)'
files:
- crates/cshop-core/src/paint.rs
- crates/cshop-core/src/path.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-core/src (paint, path)
type: Module
---

### What it does

`paint.rs` implements brush stamping and stroke accumulation, separating flow (paint per dab) from opacity (ceiling for the whole stroke) so overlapping dabs within one stroke build up rather than darken. `path.rs` defines cubic Bézier paths and boolean operations on them, represented as signed distance fields so combining shapes is cheap and remains editable.

### Public interface

**paint.rs:**
- `Brush { size, hardness, opacity, flow, spacing }` — brush shape and dynamics
- `Brush::falloff(d: f32) -> f32` — coverage at distance from centre
- `Brush::step() -> f32` — distance between dabs
- `Clip { selection, offset }` — selection bounding a stroke
- `PaintMode` enum: `Paint`, `Erase`
- `StrokeSource` enum: `Solid(Rgba8)`, `Clone { pixels, offset }`
- `Stroke::new(width, height, brush, mode, color)` — begin a stroke
- `Stroke::with_source(width, height, brush, mode, source)` — stroke from custom source
- `Stroke::add_point(p: Vec2)` — add pointer sample
- `Stroke::commit(pixels, clip) -> IRect` — apply to layer
- `Stroke::commit_to_patch(pixels, clip) -> Option<(IRect, PixelBuffer)>` — for undo
- `Stroke::render_region(snapshot, dst, rect, clip)` — live preview
- `Stroke::render_region_into_mask(snapshot, dst, rect, clip)` — paint masks
- `Stroke::take_recent() -> IRect` — changed region since last call
- `Stroke::bounds() -> IRect`, `mode()`, `brush()`, `is_empty()`
- `Stroke::set_clone_offset(offset)` — move clone source

**path.rs:**
- `Anchor { at, in_handle, out_handle }` — path node with control points
- `Anchor::corner(at)`, `smooth(at, out_handle)` — constructors
- `Anchor::is_smooth() -> bool` — whether handles are mirrored
- `SubPath { anchors, closed }` — run of anchors
- `SubPath::open(anchors)`, `closed(anchors)`, `polygon(points)` — constructors
- `SubPath::segments() -> Vec<(Vec2, Vec2, Vec2, Vec2)>` — cubic segments
- `SubPath::flatten(tolerance) -> Vec<Vec2>` — points within tolerance of curve
- `BoolOp` enum: `Union`, `Subtract`, `Intersect`, `Exclude`
- `BoolOp::combine(a: f32, b: f32) -> f32` — combine signed distances
- `BoolOp::name()`, `all()`
- `PathPart { subpaths, op }` — operand with its boolean mode
- `PathShape { parts }` — composite shape from multiple operands
- `PathShape::new(subpaths)` — constructor
- `PathShape::anchors()` — all nodes for editing
- `PathShape::bounds() -> Option<Rectf>` — including handle reach
- `PathShape::flatten(tolerance) -> Flattened` — preflattened for rasterising
- `PathShape::is_empty()`, `translate(by)`

### Key invariants

**paint.rs:**
- A stroke's coverage buffer is document-sized but only conceptually; operations treat coordinates as layer-local and clipping handles the offset.
- Flow accumulation uses `1 - prev * (1 - prev) * dab * flow` so overlapping dabs never exceed 1.0 coverage.
- Opacity is applied once during commit, not per-dab, so a slow drag does not darken.
- `take_recent()` returns and clears the region painted since the last call; coverage only increases, so incremental preview with snapshot re-renders is idempotent.
- A clone source's alpha is multiplied into coverage, so transparent source deposits nothing.

**path.rs:**
- Handles are absolute positions, not offsets from the anchor, so pen tool edits are direct.
- A closed subpath is flattened back to its start point so winding tests see no gap.
- Anchors' smoothness is determined by whether in/out handles are mirrored; a pen tool maintains this while dragging and breaks it explicitly at corners.
- Boolean operations on distance fields preserve the sign (inside/outside) exactly and approximate only magnitude near seams; operands stay in the layer and are evaluated together, so the operation remains editable.
- `bounds()` includes handle reach because curves bow outside their anchors; the control hull is a safe upper bound.

### Non-obvious decisions

**paint.rs:**
- Coverage is accumulated in a separate buffer and committed once, rather than painting each dab directly onto the layer. This is necessary because overlapping dabs must not darken a translucent stroke; opacity applies to the whole stroke, not per-dab. A dabs-directly approach would composite each dab separately and violate this.
- `clip.offset` is stored separately from the selection rather than baking it into the selection. A document-sized coverage mask is megabytes; copying one per stroke would be the most expensive operation during painting, so it is borrowed and the offset is applied per-pixel during rendering.
- Clone stamping shares the entire stroke machinery (size, hardness, opacity, flow, spacing) with the brush rather than implementing a separate tool. The only difference is the source of colour; reusing the accumulation and commit logic avoids duplicating the stroke buffer and dab interpolation.

**path.rs:**
- Handles are stored as absolute positions, not as offsets from anchors. A pen tool manipulates handles directly by dragging, so absolute coordinates avoid a conversion on every movement. The trade-off is that translating an anchor requires updating three coordinates instead of one, but that is rare compared to handle edits during drawing.
- Boolean operations are defined on signed distances rather than performing geometric intersection. Curve-curve intersection and the resulting topology is intricate and error-prone; combining fields via `min`, `max`, and negation is simple and keeps operands in the layer so the shape remains editable after the operation. The cost is that the result is not a single path but a composite that is evaluated at render time.
- Flattening is done once up front before rasterising, rather than during rasterisation per pixel. Cubic flatness testing is expensive; doing it once and caching the flattened points avoids re-subdividing the same curve many times.

### Unclear intent

**paint.rs:**
- `Stroke::take_recent()` resets `recent` to `EMPTY` each call. The docstring explains this allows a live preview to re-render only new work per frame, but the mechanism for *using* that incremental region (outside this module) is not visible in the provided code. The test `incremental_rendering_matches_a_single_commit` validates that it works, but the actual editor integration would be in the UI layer.
