---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-core/src/filters (render)'
files:
- crates/cshop-core/src/filters/render.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-core/src/filters (render)
type: Module
---

### What it does

This module provides procedural image generation filters that create images from scratch rather than transforming existing ones. It implements cloud and fiber texture generation using Perlin-like noise techniques, with customizable colors, scales, and seeds.

### Public interface

```rust
pub fn clouds(
    src: &Plane,
    scale: f32,
    seed: u64,
    foreground: Rgba8,
    background: Rgba8,
    difference: bool,
) -> Plane

pub fn fibers(
    src: &Plane,
    strength: f32,
    length: f32,
    seed: u64,
    foreground: Rgba8,
    background: Rgba8,
) -> Plane
```

### Key invariants

- All noise functions produce values in the range [0.0, 1.0] after clamping or normalization.
- Smoothstep interpolation (3t² - 2t³) is applied in `value_noise` to avoid visible grid artifacts at lattice boundaries.
- The `clouds` filter with `difference: true` preserves the alpha channel of the source image while computing color differences against its premultiplied RGB values.
- Both generators produce fully opaque output (alpha = 1.0) in normal mode.

### Non-obvious decisions

- **Smoothstep interpolation**: The cubic Hermite curve `tx * tx * (3.0 - 2.0 * tx)` is used instead of linear interpolation to eliminate visual grid patterns that would appear at lattice boundaries in value noise.
- **XOR seed variation per octave**: Each FBM octave XORs the base seed with `octave as u64 * 0x9E37` rather than incrementing or using a hash. This ensures different noise patterns per octave while maintaining determinism.
- **Premultiplied alpha handling in difference mode**: The code unpremultiplies RGB by dividing by alpha before computing the difference, then applies the original alpha to the result. This is necessary to produce correct difference blending semantics with premultiplied alpha.
- **High horizontal, low vertical frequency in fibers**: The horizontal coordinate is multiplied by 0.6 while vertical uses `vertical_step`, creating anisotropic noise that naturally produces vertical streaks rather than isotropic blobs.

### Unclear intent

- The specific choice of `0x9E37` as the XOR constant for octave seed variation is not explained; it appears to be an arbitrary large odd constant chosen to decorrelate octaves, but no derivation or reference is documented.
