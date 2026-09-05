---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates'
files:
- crates/cshop-gpu/examples/bench.rs
- crates/cshop-gpu/examples/probe.rs
- crates/cshop-ui/examples/newlayer.rs
tags:
- module
timestamp: '2026-08-30'
title: crates
type: Module
---

### What it does
Provides example programs demonstrating GPU compositor performance measurement, GPU capability detection, and UI layer manipulation in the C-Shop editor.

### Public interface

**bench.rs**
- `fn build(width: u32, height: u32, layers: usize) -> Document` — constructs a test document with multiple raster layers using varying blend modes
- `fn time(label: &str, iters: u32, f: impl FnMut()) -> f64` — measures execution time of a closure over multiple iterations, returns milliseconds per iteration

**probe.rs**
- Entry point calls `GpuContext::headless()` and prints adapter metadata

**newlayer.rs**
- `CShopApp::new(gpu)` — initializes the application
- `CShopApp::open_document(doc)` — loads a document
- `CShopApp::dispatch(Action)` — dispatches UI actions like `Action::NewLayer`
- `CShopApp::begin_stroke(pos, mode)` / `end_stroke()` — paint stroke lifecycle
- `app.doc()` — accesses current document view

### Key invariants

- `bench.rs` skips test configurations that exceed the GPU's texture memory budget to prevent allocation failures
- `newlayer.rs` verifies that a stroke modifies pixel data by checking the center pixel after `end_stroke()`
- Layer IDs allocated via `doc.tree.alloc_id()` remain valid for the document lifetime

### Non-obvious decisions

- **bench.rs untimed warmup pass**: The `time()` function executes one iteration before starting the timer to exclude shader compilation and allocation overhead from measurements, ensuring only steady-state performance is reported
- **bench.rs dirty rectangle test**: Uses a fixed 256×256 pixel dab positioned at canvas center rather than varying locations, establishing a consistent small-region baseline for comparison against full-canvas compositing

### Unclear intent

- `newlayer.rs` prints `view.doc.effective_edit_target()` without asserting or validating the result — the purpose of this diagnostic output is not clear from context
