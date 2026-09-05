---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-gpu/src (compositor, context,
  layers, lib, readback)'
files:
- crates/cshop-gpu/src/compositor.rs
- crates/cshop-gpu/src/context.rs
- crates/cshop-gpu/src/layers.rs
- crates/cshop-gpu/src/lib.rs
- crates/cshop-gpu/src/readback.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-gpu/src (compositor, context, layers, lib, readback)
type: Module
---

### What it does

Manages GPU device initialization, compositing of layer trees into final images, caching of layer textures, and conversion of working-format composites to display format. The module implements ping-pong scratch rendering with tiling to keep VRAM bounded regardless of document size.

### Public interface

**GpuContext**
- `new(instance: wgpu::Instance, surface: Option<&wgpu::Surface>) -> Result<Self, GpuError>` — Creates a GPU context from an instance, selecting an adapter suitable for the given surface.
- `headless() -> Result<Self, GpuError>` — Creates a headless context for CLI and testing.
- `work_format() -> wgpu::TextureFormat` — Returns the texture format used for compositor intermediates.
- `max_texture_dim() -> u32` — Largest single dimension a texture can have on this GPU.
- `texture_budget() -> u64` — Conservative VRAM budget in bytes for layer textures.
- `adapter_name() -> String` — Human-readable adapter name for UI display.
- `wait()` — Blocks until all GPU work completes.

**Compositor**
- `new(ctx: &GpuContext) -> Self` — Initializes compositing pipelines and scratch memory.
- `composite(ctx: &GpuContext, doc: &Document, cache: &LayerTextures, dest: &GpuTexture, region: IRect)` — Composites a region of a document into a full-document texture, tiling if necessary.
- `present(ctx: &GpuContext, src: &GpuTexture, dest: &GpuTexture)` — Converts working-format composite to 8-bit sRGB premultiplied for display.
- `last_pass_count: u32` — Render passes issued in the last composite, for status display.

**LayerTextures**
- `new() -> Self` — Creates an empty texture cache.
- `pixels(id: LayerId) -> Option<&GpuTexture>` — Retrieves cached layer pixels.
- `mask(id: LayerId) -> Option<&GpuTexture>` — Retrieves cached layer mask.
- `clear()` — Drops all cached textures.
- `over_budget() -> Option<(needed, budget)>` — Signals when document exceeds GPU memory budget.
- `memory_bytes() -> u64` — Approximate VRAM currently in use by cached textures.
- `required_bytes(doc: &Document) -> u64` — Total VRAM needed if every layer were resident.

**GpuTexture** (from texture module)
- `render_target(ctx: &GpuContext, label: &str, w: u32, h: u32, format: wgpu::TextureFormat) -> Self` — Creates a renderable texture.
- `sampled(ctx: &GpuContext, label: &str, w: u32, h: u32, format: wgpu::TextureFormat) -> Self` — Creates a readable texture.
- `write(ctx: &GpuContext, data: &[u8], size: usize)` — Uploads data to texture.

### Key invariants

- **Document-to-cache binding**: `LayerTextures.doc` must match the document ID of any layer IDs being queried, since layer IDs reset per document.
- **Scratch buffer coherence**: At each nesting depth, exactly one of the ping-pong pair (`front_is_b` flag) holds the current composite result; the other is the write target for the next pass.
- **Region isolation**: Scratch buffers are sized to the dirty region (rounded up to `SCRATCH_GRANULARITY`, capped at `MAX_TILE`), so tiling remains efficient and previous frames' debris cannot leak into composites.
- **Clip buffer separation**: Clipping base passes write to a separate clip buffer, not the ping-pong, so the base layer's alpha is captured before any subsequent passes modify it.
- **Pass-through inlining**: Groups with `BlendMode::PassThrough`, full opacity, no mask, and no clipping contribute their children directly to the parent's pass sequence with no intermediate texture.
- **Nesting depth tracking**: `Plan.max_depth` is the deepest scratch level needed for a tile; all shallower depths are allocated and cleared.
- **Budget enforcement**: Once `LayerTextures.over_budget` is set, no more textures are allocated; the UI displays the shortfall rather than showing a partially rendered image.

### Non-obvious decisions

- **Ping-pong rendering over fixed-function blending**: Fixed-function GPU blending cannot correctly express Multiply, Overlay, or Luminosity blend modes together with alpha compositing. Each layer is rendered as a full-region pass that samples the backdrop and writes the composited result, requiring two scratch textures to alternate as source and target. The shader handles blending instead of the GPU's blend unit.

- **Scratch buffers capped at `MAX_TILE`**: Unbounded scratch allocation would scale with document size (a 24 MP canvas needs gigabytes per nesting level). Capping at 2048×2048 keeps scratch memory constant across nesting depth while tiling the region, trading per-pass overhead (which is small) for bounded VRAM.

- **Separate clip buffer for clipping bases**: Capturing the clipping base into `scratch[depth].clip` rather than snapshotting the running composite preserves the correct semantics: the clipping group clips to the base layer's own alpha, not to accumulated alpha from layers below (which would make an opaque layer below disable clipping entirely).

- **Interior mutability for ping-pong flip**: The `flip()` method uses an unsafe raw pointer to mutate `front_is_b` while render passes hold shared borrows of both scratch textures. This is safe because compositing is single-threaded and only this one `bool` changes, avoiding the need to redesign the borrow structure.

- **Conservative texture budget by adapter class**: wgpu provides no direct VRAM query, so budget is estimated from adapter type (discrete GPU: 2 GB, integrated: 1 GB, other: 512 MB). This is deliberately conservative to leave room for compositor scratch, swapchain, and other GPU allocations.

- **Work format negotiated at `GpuContext` construction**: The compositor intermediate format is chosen once at device creation rather than per-composite because it must be compatible with the adapter's capabilities and consistent across all textures. Rgba16Float is used instead of Rgba16Unorm (which has better precision) because wgpu does not support Rgba16Unorm as a render target, only as a storage texture.

- **Layer tree flattened into a `Plan` before rendering**: The pass sequence is pre-computed into a `Plan` struct before any GPU commands are issued. This keeps recursion, group inlining rules, and clipping logic in one readable place and makes the pass list easy to assert on in tests, rather than interleaving planning and rendering.
