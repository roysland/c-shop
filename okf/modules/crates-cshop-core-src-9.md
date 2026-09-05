---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-core/src (pixels, resample)'
files:
- crates/cshop-core/src/pixels.rs
- crates/cshop-core/src/resample.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-core/src (pixels, resample)
type: Module
---

### What it does

`pixels.rs` provides `PixelBuffer`, a tightly-packed RGBA8 image buffer that serves as the authoritative CPU-side storage for layer pixels, with support for region operations and downscaling for thumbnails. `resample.rs` implements image transformation and resizing with multiple reconstruction filters, using premultiplied alpha to avoid dark halos around soft edges.

### Public interface

**pixels.rs:**
- `PixelBuffer::new(width: u32, height: u32) -> Self` — create a fully transparent buffer
- `PixelBuffer::filled(width: u32, height: u32, color: Rgba8) -> Self` — create a buffer filled with a color
- `PixelBuffer::from_pixels(width: u32, height: u32, data: Vec<Rgba8>) -> Option<Self>` — wrap existing pixels
- `PixelBuffer::from_rgba_bytes(width: u32, height: u32, bytes: &[u8]) -> Option<Self>` — construct from interleaved RGBA bytes
- `width() -> u32`, `height() -> u32`, `bounds() -> IRect` — dimensions
- `pixels() -> &[Rgba8]`, `pixels_mut() -> &mut [Rgba8]` — raw access
- `as_bytes() -> &[u8]` — interleaved RGBA bytes
- `row(y: u32) -> &[Rgba8]`, `row_mut(y: u32) -> &mut [Rgba8]` — single row access
- `get(x: i32, y: i32) -> Rgba8`, `set(x: i32, y: i32, c: Rgba8)` — pixel access with out-of-bounds handling
- `fill(color: Rgba8)`, `fill_rect(rect: IRect, color: Rgba8)` — fill operations
- `copy_rect(rect: IRect) -> PixelBuffer` — extract a region, padding outside with transparency
- `paste(src: &PixelBuffer, x: i32, y: i32)` — composite another buffer
- `opaque_bounds() -> IRect` — tight bounding box of non-transparent pixels
- `downscale(dst_w: u32, dst_h: u32) -> PixelBuffer` — box-filtered downscale for thumbnails
- `const SAMPLES_PER_CELL: u32` — controls downscale sampling density
- `fn sample_step(span: u32) -> u32` — stride to keep sampling under budget

**resample.rs:**
- `enum Resampling` — `Nearest`, `Bilinear`, `Bicubic`, `Lanczos3`
- `Resampling::name() -> &'static str`, `Resampling::ALL` — enumeration utilities
- `fn transform(src: &PixelBuffer, offset: (i32, i32), matrix: Transform, filter: Resampling, clip: Option<IRect>) -> Option<(PixelBuffer, (i32, i32))>` — apply affine transform
- `fn resize(src: &PixelBuffer, width: u32, height: u32, filter: Resampling) -> PixelBuffer` — resize to exact dimensions

### Key invariants

- **PixelBuffer stride**: rows are always contiguous with stride `width * 4` bytes; data length is always `width * height`
- **Out-of-bounds safety**: `get()` returns `Rgba8::TRANSPARENT`; `set()` silently drops writes; this keeps sampling loops branch-light at edges
- **Copy-rect padding**: `copy_rect()` always returns exactly `rect`-sized output, padding outside regions with transparency — undo snapshots depend on this
- **Premultiplied filtering**: all resample operations work on premultiplied alpha and un-premultiply at the end to prevent dark halos
- **Downscale cost model**: downscale uses a fixed sample budget per cell (`SAMPLES_PER_CELL`) to keep cost proportional to output size, not input size
- **Transform invertibility**: `transform()` returns `None` if the matrix is singular or produces an empty result; clip bounds prevent allocating huge buffers from wild handle drags
- **Separable resize**: horizontal and vertical passes are independent, allowing area-aware filter support to widen during reduction without 2D kernel explosion

### Non-obvious decisions

- **Downscale uses sRGB averaging, not linear**: thumbnails deliberately average in sRGB space to match what the canvas displays, rather than converting to linear for mathematical purity. This is a deliberate choice for visual consistency.
- **Two separate downscale implementations**: `PixelBuffer::downscale()` (fast, fixed-sample) is used for thumbnails regenerated every frame, while `resize()` (area-aware, slower) is for user-initiated operations. The fast path exists because a 10000x10000 document was spending 150 ms per stroke building a 48-pixel thumbnail that wasn't being examined closely; the sampling cost was proportional to the canvas rather than the output.
- **Resize uses separable 1D passes instead of 2D**: computing a full 2D kernel would require the support radius to scale with both axes simultaneously; splitting into horizontal then vertical keeps complexity manageable while still being area-aware.
- **Transform samples at pixel centres and maps backward**: rather than forward-mapping source pixels to destination, each destination pixel is mapped back through the inverse matrix and sampled once. This is correct for rotation/perspective and fast enough to run interactively, whereas forward-mapping can leave holes.
- **Degenerate resize kernels fall back to nearest**: if the filter weight calculation produces an empty or near-zero kernel, rather than leaving a transparent stripe, the code falls back to the nearest source pixel. This prevents artifacts when the reduction factor is extreme.

### Unclear intent

- The threshold `1e-6` used in multiple places (`Lanczos3` weight calculation, `Premul::to_rgba8` alpha check, `sample()` total weight) appears to guard against denormals or numerical instability, but the specific value is not documented and its relationship to f32 precision or perceptual thresholds is unclear.
