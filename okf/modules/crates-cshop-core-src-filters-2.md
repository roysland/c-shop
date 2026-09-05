---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-core/src/filters (mod, plane)'
files:
- crates/cshop-core/src/filters/mod.rs
- crates/cshop-core/src/filters/plane.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-core/src/filters (mod, plane)
type: Module
---

### What it does

Implements spatial image filters that read pixel neighbourhoods, running on the CPU with parallel processing across rows. Provides a filter system with 30+ different effects organized by category, applying changes destructively to layers while supporting preview-at-scale for interactive use.

### Public interface

```rust
pub enum Category { Blur, Sharpen, Noise, Distort, Pixelate, Render, Stylize, Other }
impl Category {
  pub fn name(self) -> &'static str
  pub const ALL: [Category; 8]
}

pub struct FilterContext {
  pub foreground: Rgba8
  pub background: Rgba8
}

pub enum Filter {
  GaussianBlur { radius: f32 }
  BoxBlur { radius: f32 }
  MotionBlur { angle: f32, distance: f32 }
  // ... 27 more variants
}
impl Filter {
  pub fn name(&self) -> &'static str
  pub fn category(&self) -> Category
  pub fn has_settings(&self) -> bool
  pub fn is_generative(&self) -> bool
  pub fn all_defaults() -> Vec<Filter>
  pub const IDENTITY_KERNEL: [f32; 25]
  pub fn apply(&self, src: &PixelBuffer, ctx: &FilterContext) -> PixelBuffer
  pub fn apply_plane(&self, src: &Plane, ctx: &FilterContext) -> Plane
  pub fn support(&self) -> Option<u32>
  pub fn scaled(&self, scale: f32) -> Filter
}

pub struct Plane {
  pub width: u32
  pub height: u32
  pub data: Vec<f32>  // premultiplied RGBA, row-major
}
impl Plane {
  pub fn new(width: u32, height: u32) -> Plane
  pub fn from_pixels(src: &PixelBuffer) -> Plane
  pub fn to_pixels(&self) -> PixelBuffer
  pub fn index(&self, x: i32, y: i32) -> usize
}
```

### Key invariants

- Filter output dimensions always match input dimensions.
- All pixel data remains finite (no NaN) after filtering.
- Filters are deterministic: applying the same filter twice to the same input produces identical results.
- Premultiplied blending prevents colour bleeding into transparent areas during neighbourhood operations.
- Float values stay unbounded during intermediate steps (sharpening/embossing overshoots), clamping only at final conversion to u8.
- Generative filters (Clouds, Fibers) produce identical results regardless of input content when seed is fixed, enabling preview-independent generation.
- Spatial filters that don't move pixels (sharpen, median, noise) preserve zero alpha values.
- Edge pixels beyond image bounds clamp to the boundary value.

### Non-obvious decisions

- **Premultiplied float buffers**: Filters operate on premultiplied `f32` planes rather than direct `u8` pixels. This prevents dark halos around soft edges during blurring and allows sharpening/embossing to overshoot without intermediate clamping flattening the effect. The conversion cost is justified on cheap filters by avoiding intermediate quantization loss.

- **Scaled filter support for preview workflow**: The `support()` and `scaled()` methods enable a two-pass preview system: render at reduced resolution, then scale parameters proportionally. Generative filters and those anchored to image extents return `None` for support, making them unbounded in scope and unable to preview from crops. This design trades preview accuracy for correctness on effects whose output depends on global properties.

- **Row-major rayon parallelization**: Filters parallelize across rows (not tiles or pixels) because blurring and distortions typically need sequential access patterns; row-level granularity amortizes rayon's thread-spawning overhead while maintaining cache locality.

- **Separate `apply_plane()` method**: Allows chaining multiple filters without re-encoding between each step, saving round-trip conversion cost when filters run in sequence.

### Unclear intent

None identified. Filter semantics are established through extensive unit tests (40+ test cases covering edge behaviour, determinism, and correctness). Module imports from other parts of the codebase (e.g., `crate::pixels::PixelBuffer`, `crate::color::Rgba8`) are external dependencies documented in their respective modules.
