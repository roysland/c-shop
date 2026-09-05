---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-core/src (selection)'
files:
- crates/cshop-core/src/selection.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-core/src (selection)
type: Module
---

### What it does

Manages 8-bit coverage masks representing pixel selections, stored only in regions where they have non-zero coverage rather than document-sized buffers. Provides construction from shapes (rectangles, ellipses, polygons), boolean combination (replace/add/subtract/intersect), modifications (feather/expand/contract/border/smooth), and marching-ants outline tracing.

### Public interface

```rust
pub enum SelectionMode { Replace, Add, Subtract, Intersect }
impl SelectionMode {
    pub fn name(self) -> &'static str
    pub fn from_modifiers(shift: bool, alt: bool) -> SelectionMode
}

pub struct Selection {
    pub fn all(width: u32, height: u32) -> Self
    pub fn empty(width: u32, height: u32) -> Self
    pub fn from_mask(mask: MaskBuffer) -> Self
    pub fn from_window(mask: MaskBuffer, window: IRect, size: (u32, u32)) -> Selection
    pub fn from_rect(width: u32, height: u32, rect: Rectf, antialias: bool) -> Self
    pub fn from_ellipse(width: u32, height: u32, rect: Rectf, antialias: bool) -> Self
    pub fn from_polygon(width: u32, height: u32, points: &[Vec2], antialias: bool) -> Self
    pub fn from_mask_bounded(mask: MaskBuffer, bounds: IRect) -> Self
    pub fn width(&self) -> u32
    pub fn height(&self) -> u32
    pub fn bounds(&self) -> IRect
    pub fn is_empty(&self) -> bool
    pub fn is_everything(&self) -> bool
    pub fn coverage(&self, x: i32, y: i32) -> u8
    pub fn window(&self) -> (&MaskBuffer, IRect)
    pub fn widen_to_document(&mut self)
    pub fn to_mask(&self) -> MaskBuffer
    pub fn memory_bytes(&self) -> u64
    pub fn combine(&mut self, other: &Selection, mode: SelectionMode)
    pub fn invert(&mut self)
    pub fn feather(&mut self, radius: f32)
    pub fn expand(&mut self, pixels: u32)
    pub fn contract(&mut self, pixels: u32)
    pub fn border(&mut self, pixels: u32)
    pub fn smooth(&mut self, radius: u32)
    pub fn contours(&mut self) -> &[Vec<Vec2>]
    pub fn dropped_contours(&self) -> usize
    pub fn compress(&self) -> CompressedSelection
    pub fn mask_mut(&mut self) -> &mut MaskBuffer
    pub fn window_origin(&self) -> (i32, i32)
    pub fn invalidate(&mut self)
}

pub struct Rectf {
    pub x0: f32, pub y0: f32, pub x1: f32, pub y1: f32
    pub fn from_points(a: Vec2, b: Vec2) -> Rectf
    pub fn pixel_bounds(&self) -> IRect
    pub fn constrain_square(from: Vec2, to: Vec2) -> Rectf
    pub fn snap_to_pixels(self) -> Rectf
    pub fn from_center(centre: Vec2, corner: Vec2) -> Rectf
}

pub struct CompressedSelection {
    pub fn restore(&self) -> Selection
    pub fn memory_bytes(&self) -> u64
}

pub(crate) fn distance_field(mask: &MaskBuffer, w: u32, h: u32, from_outside: bool) -> Vec<f32>
```

### Key invariants

- A selection is stored only where coverage is non-zero; everything outside `window` reads as unselected (0 coverage).
- `bounds` is always the bounding box of all non-zero coverage in the mask, and is always contained within `window`.
- `window` sits at document coordinates (`window.x0`, `window.y0`) and the mask is indexed relative to that origin.
- Coverage values are 8-bit (0–255), where 0 is unselected and 255 is fully selected; values 1–254 represent antialiased or feathered edges.
- The mask size always matches `(window.width(), window.height())`, never the document size.
- `contours` is `None` until first requested, then cached until the selection is mutated.
- Operations like feather, expand, and contract only modify pixels within a bounded region around `bounds` plus reach distance.

### Non-obvious decisions

- **Coverage stored as window, not document**: The mask is sized to `window` rather than the full document and stored only where coverage is non-zero. This makes a small selection in a large document cost its own area, not the canvas size. Multiple accessors (`window()`, `to_mask()`) expose this duality to callers with different needs.

- **Antialiasing via overlap calculation**: Rectangle and ellipse selections compute pixel coverage by calculating the geometric overlap between pixel squares and the shape boundary, rather than supersampling or approximating. This gives exact antialiasing at no performance cost for rectangles and exact results for ellipses at fixed supersampling cost.

- **Polygon fill via accumulated crossings**: The polygon rasterizer accumulates coverage samples per scanline at sub-pixel offsets rather than tracing individual pixels. The crossings are sorted and walked in pairs, and only pixels within the pair span are visited—avoiding the full-width walk that made large lasso selections expensive.

- **Boolean operations optimize walk region**: `combine()` only walks the region where the result can differ—Add walks the other's bounds, Subtract and Intersect walk the intersection—rather than the full canvas. This keeps the cost proportional to the selection size, not the canvas.

- **Feather/morph confined to reachable region**: Operations that move the selection edge or blur it run only within `within_reach()`, a rectangle that bounds the starting selection plus the maximum distance the operation can spread coverage. This prevents feathering a corner of a 10000×10000 document from scanning the whole image.

- **Three-pass box blur approximates Gaussian**: Feathering uses three box blurs in sequence with a computed radius to approximate a Gaussian, which runs in time independent of radius (unlike direct Gaussian convolution).

- **Exact distance via two-pass parabola algorithm**: Morphing (expand/contract) uses Felzenszwalb and Huttenlocher's parabola algorithm to compute exact Euclidean distance fields in linear time, ensuring expand and contract are symmetric and diagonals do not look visibly lopsided.

- **Contour caching with dropped-outline fallback**: Marching-ants outlines are cached and only recomputed on invalidation. If there are too many islands to draw at frame rate, later ones are dropped and `dropped_contours` reports how many, rather than simplifying or blocking.

- **Compressed selection stores only bounds region**: The undo snapshot holds only the bounding rectangle of coverage, not the full document mask. Restoring directly sets the window to that rectangle, avoiding the expensive step of painting into a document-sized buffer and trimming it back.

- **Coverage-based boolean semantics for antialiased edges**: Add uses max (union that keeps stronger coverage), Subtract scales by complement `(a*(255-b))/255`, and Intersect multiplies `(a*b)/255`. This keeps edges smooth through repeated operations instead of hardening partial coverage.

- **Horizontal blur parallelizes at row granularity**: `box_blur_h` uses `rayon` to parallelize rows independently since they are contiguous and independent, with each thread handling a row chunk.

- **Vertical blur uses banded parallelization**: `box_blur_v` splits into vertical bands (64 rows each) that each maintain independent running sums per column, so reads/writes stay sequential and bands run on separate cores. A naive column-at-a-time approach caused cache misses because each iteration would step a whole row through memory.
