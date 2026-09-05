---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-core/src (shape, snapshot,
  text)'
files:
- crates/cshop-core/src/shape.rs
- crates/cshop-core/src/snapshot.rs
- crates/cshop-core/src/text.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-core/src (shape, snapshot, text)
type: Module
---

### What it does

Provides vector shape rendering, text layout and rasterization, and efficient snapshot capture for undo/redo. Shapes are rendered via signed distance functions to rasters; text is laid out with support for wrapping and alignment; snapshots capture tile-by-tile only where strokes actually paint.

### Public interface

**shape.rs**
- `enum ShapeKind` — rectangle, ellipse, polygon, star, line, or path
- `struct ShapeContent` — geometry, size, and style
- `struct ShapeStyle` — fill, stroke, stroke alignment, antialiasing
- `enum StrokeAlign { Inside, Center, Outside }`
- `fn rasterize(content: &ShapeContent) -> Option<Rasterized>` — render to pixels
- `fn outline(kind: &ShapeKind, size: (f32, f32)) -> Vec<SubPath>` — extract contours for boolean ops
- `struct Rasterized { pixels: PixelBuffer, anchor: (i32, i32) }`

**snapshot.rs**
- `trait Grid` — abstraction over PixelBuffer and MaskBuffer
- `struct Snapshot<T>` — holds original pixels/mask values
- `fn capture<G: Grid>(&mut self, source: &G, rect: IRect)` — capture tile-by-tile on first touch
- `fn at(&self, x: i32, y: i32) -> T` — retrieve original value
- `fn restore(&self, dst: &mut PixelBuffer, rect: IRect)` — put originals back
- `fn copy_rect(&self, rect: IRect) -> PixelBuffer` — extract original rectangle

**text.rs**
- `struct TextContent { text: String, style: TextStyle, wrap_width: Option<f32> }`
- `struct TextStyle` — family, size, color, bold, italic, alignment, leading, tracking
- `enum TextAlign { Left, Center, Right }`
- `struct Layout { lines: Vec<Line>, width: f32, height: f32, ascent: f32, line_height: f32 }`
- `struct Line { glyphs: Vec<PlacedGlyph>, width: f32, baseline: f32, range: Range<usize> }`
- `fn layout(content: &TextContent, font: &FontVec) -> Layout` — compute glyph positions

### Key invariants

**shape.rs**
- A shape's raster is anchored to its box's top-left corner, regardless of stroke overhang, so layer repositioning never requires knowledge of stroke geometry.
- Signed distance determines both fill (negative) and stroke (band around zero) simultaneously, keeping them perfectly registered.
- Path flatness is 0.05 pixels to prevent thinning artifacts from chord approximation.

**snapshot.rs**
- Tiles already captured are never overwritten; a second dab over the same region preserves the true original.
- The snapshot only holds regions actually touched by the stroke; untouched regions return the `outside` sentinel value.
- Tile boundaries align at multiples of 128 pixels.

**text.rs**
- Layout origin is the layout box's top-left; the text anchor (click point) is recorded relative to the raster, allowing layer movement to work without type-specific logic.
- Baselines are absolute positions within the layout space, not relative to each line.
- Tracking (letter spacing) is applied per glyph, scaled from thousandths of an em.

### Non-obvious decisions

**shape.rs**
- Signed distance functions are used instead of scanline rasterization because one distance value yields both fill and stroke with perfect registration and antialiasing as a simple clamp rather than a coverage integral.
- Stroke alignment is implemented as a band offset from the shape's outline (`(d - offset).abs() - half`) rather than rendering separate geometry, allowing inside/center/outside to differ only in band placement.
- Path flatness (0.05 pixels) is subdivided once per shape rather than per-pixel because chords sit inside the true curve, so a coarse tolerance shrinks the shape noticeably.
- An ellipse uses an iterative approximation for distance rather than exact computation because it is "correct at the outline and close enough either side" for antialiasing and modest stroke widths.

**snapshot.rs**
- Snapshots use a tiled hash map rather than copying the entire layer upfront because the latter made pressing the mouse button cost a full layer memcpy (e.g., 150 ms for a 10000×10000 canvas), whereas tiles capture cost only where the stroke actually paints.

**text.rs**
- Glyph placement is done in two passes (measure all lines, then place glyphs) because alignment depends on knowing the widest line before computing horizontal offsets.
- Faux italic is hardcoded to 0.21 (roughly 12 degrees) because this approximates the slant most upright typefaces receive when no true italic exists.
- Tracking is applied per glyph rather than only between glyphs so that kerning and tracking compose naturally.

### Unclear intent

**shape.rs**
- The file is truncated mid-function in `text.rs` (text placement loop incomplete); the intent of the full glyph placement pass cannot be verified.
