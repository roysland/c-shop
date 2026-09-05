---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-gpu/tests'
files:
- crates/cshop-gpu/tests/composite.rs
- crates/cshop-gpu/tests/present_gamma.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-gpu/tests
type: Module
---

### What it does

GPU compositor integration tests that validate rendering correctness by comparing GPU output against CPU reference implementations. Tests cover blend modes, layer compositing, masks, clipping, adjustment layers, and the display texture format used by egui.

### Public interface

`Harness::new() -> Option<Harness>` — Creates a GPU test context, returning `None` and skipping the test if no GPU is available.

`Harness::run(&mut self, doc: &Document) -> Vec<Rgba>` — Composites a document on GPU and reads back the result as floating-point RGBA values.

### Key invariants

- GPU and CPU blend mode implementations must not diverge by more than 2 levels (out of 255) per channel; intermediate buffers are `Rgba16Float`, and quantization near white can amplify to ~1.5 levels in modes like Color Burn.
- The display texture format (`DISPLAY_FORMAT`) must not be sRGB-aware, because egui's shader linearizes whatever it samples; an sRGB texture would be linearized twice, darkening the display while leaving saved files correct.
- Adjustment layers must not affect transparent areas (alpha must remain zero).
- Transparent areas outside layer bounds must stay transparent; layers do not wrap.
- Compositing a sub-region via `IRect` must leave pixels outside that region untouched in the target texture.

### Non-obvious decisions

- **Headless GPU contexts return `Option` instead of failing**: Tests gracefully skip rather than fail when no GPU is available, allowing `cargo test` to be meaningful on headless CI. This is done via `GpuContext::headless()` returning `Result` and tests returning early if `None`.
- **Adjustment layer opacity is implemented as blend transparency, not effect strength**: `adjustment_opacity_fades_the_effect` tests that opacity fades the layer itself (compositing it with reduced alpha), not that it reduces the magnitude of the adjustment. This is why inverting at 50% opacity gives mid-grey, not a partially-inverted image.
- **Clipped layers all use the same base alpha, not cascading alpha**: `two_clipped_layers_share_one_base` verifies that multiple clipped layers see the base layer's alpha independently, not the accumulated alpha after prior clipped layers. This requires clipped layers to composite into a separate buffer that is then masked by the base.
- **Premultiplication happens in gamma space for the display texture**: `presenting_a_translucent_colour_premultiplies_in_gamma` confirms that stored values are sRGB-encoded colors scaled by alpha (for egui), not linear values premultiplied then encoded. Un-premultiplying in gamma space must recover the original encoded color.

### Unclear intent

- `Harness::cache` field purpose is not evident from test code alone; it appears to cache layer textures across composite calls, but the tests do not demonstrate why a fresh cache per `run()` would be insufficient. Refer to `cshop_gpu/src` modules for cache semantics.
