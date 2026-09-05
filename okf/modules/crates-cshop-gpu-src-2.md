---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-gpu/src (texture)'
files:
- crates/cshop-gpu/src/texture.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-gpu/src (texture)
type: Module
---

### What it does

Provides a thin wrapper around wgpu textures that bundles the texture object, its view, and dimensions together. Defines format constants for compositor working buffers, display output, layers, and masks, along with convenience constructors for common texture usage patterns (render targets and sampled textures).

### Public interface

```rust
pub const WORK_FORMAT: wgpu::TextureFormat
pub const DISPLAY_FORMAT: wgpu::TextureFormat
pub const LAYER_FORMAT: wgpu::TextureFormat
pub const MASK_FORMAT: wgpu::TextureFormat

pub struct GpuTexture {
    pub texture: wgpu::Texture
    pub view: wgpu::TextureView
    pub width: u32
    pub height: u32
    pub format: wgpu::TextureFormat
}

impl GpuTexture {
    pub fn new(ctx: &GpuContext, label: &str, width: u32, height: u32, 
               format: wgpu::TextureFormat, usage: wgpu::TextureUsages) -> Self
    
    pub fn render_target(ctx: &GpuContext, label: &str, width: u32, height: u32, 
                         format: wgpu::TextureFormat) -> Self
    
    pub fn sampled(ctx: &GpuContext, label: &str, width: u32, height: u32, 
                   format: wgpu::TextureFormat) -> Self
    
    pub fn size(&self) -> wgpu::Extent3d
    
    pub fn write(&self, ctx: &GpuContext, data: &[u8], bytes_per_pixel: u32)
    
    pub fn write_region(&self, ctx: &GpuContext, data: &[u8], bytes_per_pixel: u32,
                        x: u32, y: u32, w: u32, h: u32)
}
```

### Key invariants

- Width and height are always at least 1 (enforced by `.max(1)` in `new`).
- `WORK_FORMAT` uses 16-bit floating point to prevent banding in stacked blend and adjustment layers.
- `DISPLAY_FORMAT` and `LAYER_FORMAT` are deliberately `Rgba8Unorm` (not `*Srgb`) because shaders expect raw gamma-encoded values, not hardware-linearized ones.
- Data passed to `write_region` must be exactly `w * h * bytes_per_pixel` bytes.
- `bytes_per_pixel` parameter must match the texture's format.

### Non-obvious decisions

**Display format is not sRGB-aware despite holding sRGB-encoded components.** The egui renderer expects to sample raw encoded bytes and applies its own gamma correction in the fragment shader before writing to the framebuffer. Using `*Srgb` would cause the hardware to linearize on sample *and* the shader to linearize again, darkening the canvas by two stops while leaving saved files correct.

**Minimum dimension of 1 instead of rejecting 0-sized allocations.** This silently clamps rather than failing, accommodating callers that may dynamically resize buffers to zero and back without special-case handling.
