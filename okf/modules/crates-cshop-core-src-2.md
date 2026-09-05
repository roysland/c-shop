---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-core/src (blend, color,
  curve)'
files:
- crates/cshop-core/src/blend.rs
- crates/cshop-core/src/color.rs
- crates/cshop-core/src/curve.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-core/src (blend, color, curve)
type: Module
---

### What it does

Implements layer blending modes, color representations, and editable tone curves for the C-Shop image compositor. The blend module provides 27 blend modes following W3C compositing specs with PSD compatibility; the color module handles sRGB/linear-light conversions for both 8-bit and float representations; the curve module offers monotone cubic interpolation for tone adjustments.

### Public interface

**blend.rs:**
- `enum BlendMode` – 27 layer blend modes with discriminants matching shader constants
- `BlendMode::MENU` – menu-ordered list with separator markers
- `BlendMode::all()` – iterator over real modes (excluding PassThrough)
- `BlendMode::name(self) -> &'static str`
- `BlendMode::psd_key(self) -> &'static [u8; 4]`
- `BlendMode::from_psd_key(key: &[u8; 4]) -> Option<BlendMode>`
- `BlendMode::is_non_separable(self) -> bool`
- `fn blend_channel(mode: BlendMode, cb: f32, cs: f32) -> f32`
- `fn blend_rgb(mode: BlendMode, cb: [f32; 3], cs: [f32; 3]) -> [f32; 3]`
- `fn composite(mode: BlendMode, backdrop: Rgba, src: Rgba, coverage: f32) -> Rgba`

**color.rs:**
- `struct Rgba8` – 8-bit sRGB with straight alpha
- `Rgba8::new(r: u8, g: u8, b: u8, a: u8) -> Self`
- `Rgba8::opaque(r: u8, g: u8, b: u8) -> Self`
- `Rgba8::to_f32(self) -> Rgba`
- `Rgba8::to_linear(self) -> Rgba`
- `Rgba8::from_hex(s: &str) -> Option<Rgba8>`
- `Rgba8::to_hex(self) -> String`
- `struct Rgba` – f32 straight alpha (space context-dependent)
- `Rgba::new(r: f32, g: f32, b: f32, a: f32) -> Self`
- `Rgba::to_u8(self) -> Rgba8`
- `Rgba::to_srgb8(self) -> Rgba8`
- `Rgba::luma(self) -> f32`
- `fn srgb_to_linear(c: f32) -> f32`
- `fn linear_to_srgb(c: f32) -> f32`
- `fn rgb_to_hsv(r: f32, g: f32, b: f32) -> (f32, f32, f32)`
- `fn hsv_to_rgb(h: f32, s: f32, v: f32) -> (f32, f32, f32)`

**curve.rs:**
- `struct Curve` – editable tone curve with monotone cubic interpolation
- `Curve::new(points: Vec<(f32, f32)>) -> Self`
- `Curve::points(&self) -> &[(f32, f32)]`
- `Curve::is_identity(&self) -> bool`
- `Curve::add(&mut self, x: f32, y: f32) -> usize`
- `Curve::move_point(&mut self, index: usize, x: f32, y: f32)`
- `Curve::remove(&mut self, index: usize)`
- `Curve::hit(&self, x: f32, y: f32, radius: f32) -> Option<usize>`
- `Curve::eval(&self, x: f32) -> f32`
- `Curve::to_lut(&self) -> [u8; 256]`

### Key invariants

**blend.rs:**
- `BlendMode` discriminants must never be renumbered; they are the contract with `composite.wgsl` shader code
- All 27 real modes must appear exactly once in `MENU` (separated by `None` markers)
- PSD keys must be unique and round-trip via `from_psd_key`
- Blend operations work in document space (sRGB-encoded) with straight alpha
- All channel blend functions must return finite values for all inputs in `[0, 1]`

**color.rs:**
- `Rgba8::to_f32()` and `Rgba::to_u8()` are precise inverses with no transfer function
- `Rgba8::to_linear()` and `Rgba::to_srgb8()` form a round-trip for all byte values 0–255
- Transfer functions are exact at endpoints: `srgb_to_linear(0) ≈ 0`, `srgb_to_linear(1) ≈ 1`
- `Rgba` semantics depend on call site (document space vs. linear light)
- Luma calculation uses Rec. 601 coefficients (0.30 R + 0.59 G + 0.11 B)

**curve.rs:**
- Control points are always sorted by x-coordinate
- Curve always spans the full range `[0, 1]` on both axes (endpoints pinned to x=0 and x=1)
- Minimum two control points (the endpoints)
- Interpolation is monotone cubic (Fritsch–Carlson); cannot overshoot control points
- Flat spans (zero slope) remain flat to prevent dipping between equal points

### Non-obvious decisions

**blend.rs:**
- Non-separable modes (Hue, Saturation, Color, Luminosity, DarkerColor, LighterColor) cannot be applied per-channel and are handled by `blend_rgb` while separable modes use `blend_channel`. This split is necessary because modes like Hue must consider all three RGB channels together to preserve color relationships.
- The `composite` function applies blending before source-over compositing and uses straight alpha throughout rather than premultiplied, then reconstructs straight alpha in the output. This matches established editor behavior and simplifies the W3C formula implementation.

**color.rs:**
- `Rgba` deliberately keeps no intrinsic space tag; the space is determined by the call chain (whether the caller used `to_f32()` for document space or `to_linear()` for linear light). This avoids runtime type overhead while shifting verification burden to the call site, which must be read to understand intent.
- The sRGB transfer function uses the official piecewise formula with a linear segment near black rather than a pure power curve, trading simplicity for ICC compliance.

**curve.rs:**
- Fritsch–Carlson tangent clamping is applied to prevent monotone cubic (Catmull–Rom) overshoot, which would cause bright pixels to darken visibly when dragging a curve control point upward—a classic UI bug. The per-span limiting keeps the interpolant within the convex hull of its two control points.
- Endpoints are always pinned to x=0 and x=1 even when dragged horizontally; only vertical movement is allowed. This ensures the curve always covers the full input range and simplifies the evaluation loop.

### Unclear intent

**blend.rs:**
- The `Dissolve` mode is listed but appears to have no special blending logic—`blend_channel` returns `cs` (source color) for it, same as `Normal`. Its purpose relative to `Normal` is not evident from the code alone; this may be a PSD compatibility placeholder or stochastic blend mode that is implemented elsewhere (possibly in the shader or UI layer).
