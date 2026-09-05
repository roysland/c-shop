---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-core/examples'
files:
- crates/cshop-core/examples/adjbench.rs
- crates/cshop-core/examples/filterbench.rs
- crates/cshop-core/examples/fontscan.rs
- crates/cshop-core/examples/fxrender.rs
- crates/cshop-core/examples/selbench.rs
- crates/cshop-core/examples/shaperender.rs
- crates/cshop-core/examples/textrender.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-core/examples
type: Module
---

### What it does

This module contains standalone example and benchmarking programs that demonstrate and measure the performance of core graphics operations: adjustments, filters, selections, text rendering, shape rasterization, layer effects, and font loading. Each example generates output files or benchmark results to help developers understand capability and performance characteristics.

### Public interface

Each file is a standalone `main()` program invoked via `cargo run --release -p cshop-core --example <name>`:

- `adjbench.rs`: Benchmarks `Adjustment::apply()` per-pixel vs. `Adjustment::prepare()` on 320×320 and 1600×1200 buffers.
- `filterbench.rs`: Measures throughput of all filters from `Filter::all_defaults()` on 1920×1080 and 6000×4000 images.
- `selbench.rs`: Times selection operations: rectangular/elliptical marquee, lasso polygon, outline tracing, feathering, expanding, magic wand on noisy photos.
- `textrender.rs`: Renders sample text with various styles (bold, italic, tracking, sizes, alignment, wrapping) and writes output.
- `shaperender.rs`: Renders sample shapes (rectangles, ellipses, polygons, lines) with different stroke styles and writes output.
- `fxrender.rs`: Renders layer effects (drop shadow, glow, bevel, satin, stroke, overlays) individually and in combination, writes output.
- `fontscan.rs`: Reports font database scan time, family count, and default font loading performance.

### Key invariants

- Benchmarks use `std::hint::black_box()` to prevent dead-code elimination of measured operations.
- Adjustment and filter benchmarks run on synthetic images with predictable patterns to isolate the code path being measured.
- Selection benchmarks include worst-case scenarios (wand on noisy photo with ragged boundaries, lasso with 400 points).
- Output files are written to current directory unless a path argument is provided.
- All rendering examples output raw RGBA bytes (`.raw` extension) or reference PNG format, with dimensions logged to stdout.

### Non-obvious decisions

**`adjbench.rs` demonstrates the prepared path**: The example explicitly compares per-pixel `adj.apply()` calls against `adj.prepare().apply_buffer()` to justify why the prepared variant exists. This is a teaching example showing the performance cliff that motivated the API design.

**`selbench.rs` generates synthetic photo data**: The `photo()` function creates a worst-case test image (smooth gradients with noise) specifically because the magic wand's boundary-tracing performance degrades on ragged, noisy edges—not on simple solid regions. This makes the benchmark realistic.

**`fxrender.rs` renders to a large composite sheet**: Rather than individual output files, all effect cases are composited onto a single 1240×1300 sheet in a 4-column grid. This enables visual comparison of all effects in one image without file proliferation.

### Unclear intent

None identified. All examples follow straightforward patterns: measure performance, generate diagnostic output, or render test cases. Font family properties (`has_bold`, `has_italic`) are queried directly from `FontDb` and are self-explanatory.
