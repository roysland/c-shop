---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-core/src/filters (blur,
  distort, effects)'
files:
- crates/cshop-core/src/filters/blur.rs
- crates/cshop-core/src/filters/distort.rs
- crates/cshop-core/src/filters/effects.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-core/src/filters (blur, distort, effects)
type: Module
---

### What it does

Provides image filtering operations organized into three categories: blur effects (gaussian, box, motion, radial, surface), geometric distortions (twirl, pinch, spherize, wave, polar coordinates, mosaic, crystallize), and general effects (sharpening, noise, morphology, edge detection, emboss, solarization). All filters operate on `Plane` objects representing RGBA image data.

### Public interface

**blur.rs**
- `gaussian(src: &Plane, radius: f32) -> Plane`
- `box_blur(src: &Plane, radius: f32) -> Plane`
- `motion(src: &Plane, angle_degrees: f32, distance: f32) -> Plane`
- `radial(src: &Plane, amount: f32, kind: RadialKind, centre: (f32, f32)) -> Plane`
- `surface(src: &Plane, radius: f32, threshold: f32) -> Plane`
- `average(src: &Plane) -> Plane`
- `enum RadialKind { Spin, Zoom }`

**distort.rs**
- `twirl(src: &Plane, angle_degrees: f32) -> Plane`
- `pinch(src: &Plane, amount: f32) -> Plane`
- `spherize(src: &Plane, amount: f32) -> Plane`
- `wave(src: &Plane, amplitude: f32, wavelength: f32, vertical: bool) -> Plane`
- `polar_coordinates(src: &Plane, to_polar: bool) -> Plane`
- `mosaic(src: &Plane, size: u32) -> Plane`
- `crystallize(src: &Plane, size: u32, seed: u64) -> Plane`
- `fragment(src: &Plane, distance: i32) -> Plane`

**effects.rs**
- `sharpen(src: &Plane, amount: f32) -> Plane`
- `unsharp_mask(src: &Plane, amount: f32, radius: f32, threshold: f32) -> Plane`
- `high_pass(src: &Plane, radius: f32) -> Plane`
- `add_noise(src: &Plane, amount: f32, monochromatic: bool, gaussian: bool, seed: u64) -> Plane`
- `median(src: &Plane, radius: u32) -> Plane`
- `dust_and_scratches(src: &Plane, radius: u32, threshold: f32) -> Plane`
- `morphology(src: &Plane, radius: u32, maximum: bool) -> Plane`
- `offset(src: &Plane, dx: i32, dy: i32, wrap: bool) -> Plane`
- `find_edges(src: &Plane) -> Plane`
- `emboss(src: &Plane, angle_degrees: f32, height: f32, amount: f32) -> Plane`
- `solarize(src: &Plane) -> Plane`
- `diffuse(src: &Plane, amount: u32, seed: u64) -> Plane`
- `custom(src: &Plane, kernel: &[f32; 25], divisor: f32, offset: f32) -> Plane`

### Key invariants

- All filters return a new `Plane` with the same dimensions as the input (or early-return a clone if the effect is a no-op).
- RGBA data is always 4 channels per pixel, stored as contiguous `f32` values in `Plane.data`.
- Filters using `par_chunks_mut` operate on rows in parallel; indexing into `row` uses byte offsets (`x * 4` for channel `c` at pixel `x`).
- Clamping and boundary handling preserve alpha channel integrity; geometric filters use sampling at fractional coordinates to interpolate.
- Noise generation is seeded by position (`Rng::at(seed, x, y)`) to ensure deterministic, spatially-coherent grain across parallel execution.

### Non-obvious decisions

- **Radius-to-sigma conversion in `gaussian`**: The radius parameter is divided by 3.0 before passing to `gaussian_blur`. This treats radius as "three sigma" (where the curve has effectively fallen to nothing), so a radius of 3 yields sigma of 1, not sigma 3. This prevents blur effects from being unexpectedly strong.
- **Backward mapping in distortion**: All geometric filters use backward maps (destination → source lookup) rather than forward maps. Forward mapping would leave holes where the transform stretches; backward mapping guarantees every output pixel receives a value.
- **Motion blur and radial blur sampling**: Both use a centered sampling pattern (t ranges from -0.5 to 0.5 after normalization) so the smear is centered on the original pixel rather than biased in one direction.
- **Crystallize uses local site search**: Rather than comparing every pixel to every Voronoi site in the image, it only checks the nine neighboring grid cells, since jittered sites are guaranteed to stay within their own cell.
- **Straight-alpha threshold in `unsharp_mask` and `add_noise`**: Operations compare and apply noise in straight (unpremultiplied) alpha space before re-premultiplying, ensuring that threshold values and noise amounts mean what they numerically claim even when alpha varies.
- **Round structuring element in `morphology`**: Uses a circular (Euclidean distance) structuring element rather than a square one to avoid leaving visible corners on dilated shapes.

### Unclear intent

- **`surface` blur fallback behavior**: When accumulated weight is below `1e-6`, the filter copies the center pixel unchanged rather than using the partial accumulation. It is unclear whether this guards against division-by-zero artifacts in edge cases or represents intentional preservation of isolated pixels.
