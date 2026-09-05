---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-core/src (transform, tree,
  wand)'
files:
- crates/cshop-core/src/transform.rs
- crates/cshop-core/src/tree.rs
- crates/cshop-core/src/wand.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-core/src (transform, tree, wand)
type: Module
---

### What it does

`transform.rs` provides 3×3 projective transformation matrices for affine and perspective operations like translate, scale, rotate, skew, and distort. `tree.rs` manages a hierarchical layer structure with parent-child relationships and ordering. `wand.rs` implements colour-based selection tools (Magic Wand, Grow, Similar) using flood fill and parallel pixel scanning.

### Public interface

**transform.rs:**
- `Transform::IDENTITY`, `Transform::translate(dx, dy)`, `Transform::scale(sx, sy)`, `Transform::rotate(radians)`, `Transform::skew(kx, ky)`, `Transform::about(pivot, inner)`
- `Transform::then(self, next) -> Transform` — compose transformations
- `Transform::apply(p: Vec2) -> Vec2` — map a point through the transform
- `Transform::invert() -> Option<Transform>` — compute inverse
- `Transform::from_quad(src: IRect, dst: [Vec2; 4]) -> Option<Transform>` — construct from corner mappings
- `Transform::transformed_bounds(rect: IRect) -> IRect` — axis-aligned bounds after transform
- `Transform::is_identity() -> bool`
- `enum Handle` with variants `TopLeft`, `Top`, `TopRight`, `Right`, `BottomRight`, `Bottom`, `BottomLeft`, `Left`, `Body`, `Rotate`
- `Handle::unit_position() -> Vec2`, `Handle::corner_index() -> Option<usize>`, `Handle::opposite() -> Handle`

**tree.rs:**
- `LayerTree::new()`, `LayerTree::alloc_id() -> LayerId`
- `LayerTree::get(id) -> Option<&Layer>`, `LayerTree::get_mut(id) -> Option<&mut Layer>`
- `LayerTree::children(parent: Option<LayerId>) -> &[LayerId]`, `LayerTree::root() -> &[LayerId]`
- `LayerTree::insert(layer, parent, index) -> LayerId`, `LayerTree::push(layer, parent) -> LayerId`
- `LayerTree::remove(id) -> Vec<Layer>`, `LayerTree::restore(layers, root, pos)`
- `LayerTree::position(id) -> Option<LayerPos>`, `LayerTree::move_to(id, parent, index) -> bool`
- `LayerTree::is_ancestor(ancestor, id) -> bool`, `LayerTree::ancestors(id) -> Vec<LayerId>`, `LayerTree::depth(id) -> usize`
- `LayerTree::is_effectively_visible(id) -> bool`
- `LayerTree::iter_all() -> Vec<LayerId>`, `LayerTree::visible_rows() -> Vec<(LayerId, usize)>`, `LayerTree::neighbour_after_removal(pos) -> Option<LayerId>`
- `struct LayerPos { parent: Option<LayerId>, index: usize }`

**wand.rs:**
- `struct WandOptions { tolerance: u8, contiguous: bool, antialias: bool }`
- `magic_wand(source: &PixelBuffer, seed_x, seed_y, options) -> Selection`
- `grow(source: &PixelBuffer, selection: &Selection, options) -> Selection`
- `similar(source: &PixelBuffer, selection: &Selection, options) -> Selection`

### Key invariants

**transform.rs:**
- A 3×3 matrix is applied to homogeneous points; division by w handles perspective.
- `Transform::then` composes left-to-right: `a.then(b)` means "apply a, then b".
- `Handle::CORNERS` order is top-left, top-right, bottom-right, bottom-left, matching `Transform::from_unit_quad` expectations.
- Near-zero w values are clamped to avoid NaN propagation.

**tree.rs:**
- Index 0 is the bottom of the stack; higher indices are toward the top.
- Parent and child links must always be bidirectional: a layer's `parent` field and its presence in `parent.children` must match.
- Layer IDs are never reused; stale IDs resolve to `None` rather than aliasing.
- `LayerTree::remove` returns layers in post-order (children before parents), safe for reversal.
- A group cannot be moved into itself, directly or transitively.
- `is_effectively_visible` requires the layer itself and all enclosing groups to be visible and have non-zero opacity.

**wand.rs:**
- Colour matching uses per-channel maximum difference, not Euclidean distance.
- Alpha participates in matching, preventing spills across transparent edges.
- `magic_wand` with `contiguous=false` runs in a single parallel pass over rows.
- `grow` seeds the queue with pixels already selected (coverage ≥ 128) before expanding outward.
- `similar` quantises the palette to `tolerance` step size to avoid scanning millions of colours on photographic images.

### Non-obvious decisions

**transform.rs:**
- Rotation direction is counter-clockwise in y-down space (π/2 takes +x to +y), which differs from typical math conventions but matches screen coordinates.
- `transformed_bounds` snaps floating-point noise with ±1e-3 epsilon before rounding outward. This prevents repeated 90° rotations from creeping outward by a pixel on each turn due to cosine/sine not being exactly zero at right angles in f32.
- `from_unit_quad` solves a 2×2 system for perspective terms (g, h) from the diagonal mismatch; a parallelogram skips this (sets g, h to 0) as a fast path.

**tree.rs:**
- `children_vec_mut` falls back to root when inserting under a non-group layer rather than panicking, keeping the layer reachable instead of leaking it, though this logs a warning.
- `move_to` computes the landing index before removal and then corrects downward by 1 if moving within the same parent and the old position was earlier. This compensates for the hole removal creates.
- `remove` does not immediately delete child links; it preserves the subtree structure so `restore` can re-thread it without duplication.

**wand.rs:**
- Flood fill is reduced to runs (horizontal stretches of matching pixels) rather than individual pixels. This keeps the stack shallow and avoids recursion overflow on large flat areas, which would occur with naive four-way recursion.
- The non-contiguous path uses `rayon::par_chunks_mut` over rows and reduces their coverage bounds in parallel. This amortises the per-pixel comparison cost.
- `grow` initializes the queue by scanning the selection bounds for pixels already selected (coverage ≥ 128), ensuring outward expansion starts from known interior points.

### Unclear intent

**wand.rs:**
- The `finish` function is called at the end of `magic_wand`, `grow`, and `similar` but its implementation is cut off in the provided source. Its purpose (antialiasing, coverage normalization, or mask-to-selection conversion) cannot be determined from context.
