---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-core/src (adjust)'
files:
- crates/cshop-core/src/adjust.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-core/src (adjust)
type: Module
---

### What it does

Provides a unified system for colour adjustments that work identically across CPU, GPU shaders, and adjustment layers. Each adjustment is a pure function from colour to colour, with per-channel adjustments baked into lookup tables and multi-channel adjustments computed via shader formulas.

### Public interface

```rust
pub struct LevelsChannel {
    pub input_black: f32,
    pub input_white: f32,
    pub gamma: f32,
    pub output_black: f32,
    pub output_white: f32,
}

pub enum Adjustment {
    BrightnessContrast { brightness: f32, contrast: f32 },
    Levels { rgb: LevelsChannel, channels: [LevelsChannel; 3] },
    Curves { curves: [Curve; 4] },
    Exposure { exposure: f32, offset: f32, gamma: f32 },
    Vibrance { vibrance: f32, saturation: f32 },
    HueSaturation { hue: f32, saturation: f32, lightness: f32, colorize: bool },
    ColorBalance { shadows: [f32; 3], midtones: [f32; 3], highlights: [f32; 3], preserve_luminosity: bool },
    BlackAndWhite { weights: [f32; 6], tint: Option<Rgba8> },
    ChannelMixer { matrix: [[f32; 4]; 3], monochrome: bool },
    PhotoFilter { color: Rgba8, density: f32, preserve_luminosity: bool },
    Invert,
    Posterize { levels: u32 },
    Threshold { level: f32 },
    GradientMap { stops: Vec<GradientStop> },
}

impl Adjustment {
    pub fn name(&self) -> &'static str;
    pub fn all_defaults() -> Vec<Adjustment>;
    pub fn has_settings(&self) -> bool;
    pub fn kind(&self) -> AdjustKind;
    pub fn prepare(&self) -> Prepared<'_>;
    pub fn apply(&self, c: Rgba) -> Rgba;
    pub fn apply_rgb(&self, c: [f32; 3]) -> [f32; 3];
    pub fn is_identity(&self) -> bool;
    pub fn bake_lut(&self) -> [u8; 1024];
    pub fn gpu_params(&self) -> [[f32; 4]; 4];
}

pub struct Prepared<'a> {
    adjustment: &'a Adjustment,
    lut: Option<[u8; 1024]>,
}

impl Prepared<'_> {
    pub fn apply(&self, c: Rgba) -> Rgba;
    pub fn apply_buffer(&mut buffer: &mut [Rgba8]);
    pub fn apply_rgb(&self, c: [f32; 3]) -> [f32; 3];
}

pub enum AdjustKind {
    Lut = 0,
    GradientMap = 1,
    HueSaturation = 2,
    Vibrance = 3,
    ColorBalance = 4,
    BlackAndWhite = 5,
    ChannelMixer = 6,
    PhotoFilter = 7,
}
```

### Key invariants

- Adjustments must leave alpha unchanged; they only modify colour.
- All output values must remain finite and within `[0.0, 1.0]` (with tolerance for float precision).
- The prepared path (via `Prepared::apply_buffer`) must produce identical results to calling `apply` directly, only differing in performance.
- Per-channel lookup table adjustments (LUT kind) serialize to a 256-entry RGBA table; formula-based adjustments return an identity ramp to prevent stale table contamination.
- The `AdjustKind` numbering is a contract with `composite.wgsl` and must never be renumbered.
- Neutral settings (at defaults or with parameters like `brightness=0`, `contrast=0`) must leave colours unchanged.

### Non-obvious decisions

- **Lookup table structure**: Per-channel adjustments store results as `[u8; 1024]` (256 entries × 4 channels) indexed by input level. This allows a single table fetch in the shader regardless of which adjustment is applied, maximising GPU efficiency. Formula-based adjustments return an identity ramp to prevent a stale table from tainting results if the adjustment kind changes.

- **Exposure working in linear space**: Exposure is one of the few adjustments that must leave sRGB space, apply scaling in linear light (via `2^stops`), apply gamma correction, and convert back. This is because exposure models physical camera stops, which double/halve linear intensity, not gamma-encoded values.

- **Vibrance vs saturation separation**: Vibrance is weighted toward less-saturated colours using `(1.0 - saturation)`, allowing muted tones to gain colour while preventing already-vivid colours (like skin tones) from oversaturating. Saturation is uniform across all hues.

- **Colorize clamping order**: In `hue_saturation`, colorize clamps saturation *after* halving (`((saturation + 1.0) * 0.5).clamp(0.0, 1.0)`) rather than before. Clamping first would map the entire upper half of the slider to maximum saturation, wasting its range.

- **Lightness as a lift/drop rather than scale**: Lightness moves colours toward white (positive) or black (negative) as a blend, not by scaling the value. This matches user expectation that a maximised lightness slider produces pure white regardless of the starting colour.

- **GradientMap luma calculation**: Uses `0.30 * R + 0.59 * G + 0.11 * B`, the standard Rec. 601 coefficients, for consistency with other luma-based operations in the codebase.

- **Prepared borrows rather than copies**: `Prepared<'a>` holds a reference to `Adjustment` instead of cloning it. This is cheap for formula-based adjustments (which have no table) and avoids unnecessary allocations when the caller already owns the adjustment.

### Unclear intent

- The `scale` variable computed in `black_and_white` is assigned but never used (`let scale = grey / 0.5_f32.max(1e-4); let _ = scale;`). The tint path multiplies by `grey * 2.0` directly. It is unclear whether this is dead code, a placeholder for future logic, or a deliberate normalization that was superseded.
