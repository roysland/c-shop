---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-core/tests'
files:
- crates/cshop-core/tests/fill_speedups.rs
- crates/cshop-core/tests/history_memory.rs
- crates/cshop-core/tests/path_shapes.rs
- crates/cshop-core/tests/selection_regions.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-core/tests
type: Module
---

### What it does

Integration tests for `cshop-core` that validate correctness of optimized implementations by comparing them against obvious reference implementations, and verify that memory-bounded history and region-confined selections behave correctly.

### Public interface

**fill_speedups.rs**
- `#[test] the_baked_ramp_matches_the_exact_colours()` — validates precomputed gradient color tables against computed values
- `#[test] feathering_stays_symmetric_across_the_blur_bands()` — checks feather blur doesn't introduce seams at band boundaries
- `#[test] the_flood_fill_matches_a_naive_one()` — compares row-slice flood fill against single-pixel queue implementation
- `#[test] a_gradient_renders_the_same_whatever_the_row_order()` — verifies parallel rendering produces identical output to sequential

**history_memory.rs**
- `#[test] a_flat_edit_costs_the_history_nothing()` — validates that flat color regions aren't stored pixel-by-pixel
- `#[test] undo_restores_exactly_what_was_there()` — verifies undo/redo round-trip fidelity
- `#[test] the_oldest_steps_are_dropped_once_the_budget_is_reached()` — checks memory budget enforcement and LRU eviction
- `#[test] a_single_entry_survives_a_budget_it_cannot_fit()` — ensures at least one step remains undoable regardless of budget

**path_shapes.rs**
- `#[test] a_shape_converted_to_contours_renders_as_itself()` — compares rendering of shapes as distance fields versus as Bézier outlines
- `#[test] the_boolean_operations_render_what_they_mean()` — validates union, intersect, subtract, exclude operations render correctly
- `#[test] an_open_path_is_stroked_and_not_filled()` — verifies unclosed paths stroke but don't self-fill

**selection_regions.rs**
- `#[test] feather_is_the_same_either_way()` — compares bounded feather against whole-canvas feather
- `#[test] expand_and_contract_are_the_same_either_way()` — compares bounded expand/contract against whole-canvas
- `#[test] border_is_the_same_either_way()` — compares bounded border against whole-canvas
- `#[test] smooth_is_the_same_either_way()` — compares bounded smooth against whole-canvas
- `#[test] the_bounds_are_still_correct_after_a_bounded_edit()` — validates selection bounds remain accurate after region-confined operations
- `#[test] combining_is_the_same_either_way()` — compares bounded combine against whole-canvas
- `#[test] a_small_selection_on_a_big_canvas_holds_little()` — verifies selections store coverage only where needed

### Key invariants

**Gradient rendering**: Baked color tables remain within 2 levels (out of 255) of exact computed colors; rendering produces bit-identical results regardless of row processing order.

**Selection feathering**: Blur output is symmetric about the feathered region's axis; no seams appear at band boundaries (64-row chunks).

**Flood fill**: The optimized row-slice implementation selects exactly the same pixels as a naive four-connected queue-based flood fill with identical color tolerance.

**History memory**: Total bytes tracked equals sum of all ReplacePixels steps' reported sizes; oldest steps are dropped first when budget is exceeded; one step always remains undoable; undo/redo restores exact pixel state.

**Selection storage**: Memory usage scales with selection's bounding box, not canvas size; coverage queries remain correct at any canvas coordinate including outside bounds; compress/restore cycle preserves both coverage data and memory efficiency.

**Path to shape equivalence**: A shape converted to Bézier contours and filled differs by <1% of pixels from the native distance-field rendering; boolean operations render coverage that matches the operation's logical definition; unclosed paths stroke but do not fill.

**Region-confined operations**: Feather, expand, contract, border, and smooth produce identical output whether computed over full canvas or bounded to region plus operation reach; resulting bounds accurately reflect actual coverage.

### Non-obvious decisions

**Feathering band symmetry test**: Rather than checking blur mathematics directly, the test verifies output is symmetric about a vertically-centered block. This is a cheap way to catch seams without knowing the expected blur profile — any asymmetry must be an implementation artifact (band boundary), not mathematically correct behavior.

**History memory measurement**: Flat color fills over flat backgrounds store zero bytes because the delta can be represented as "same color everywhere" rather than pixel data. Only heterogeneous content underneath forces storage of the before-state. This trades computation (checking homogeneity) for storage.

**Selection bounds forcing in region tests**: To test that bounded operations equal whole-canvas operations, the test adds specks in all four document corners. This forces the selection's bounding rect to span the canvas, making the bounded codepath take the whole-canvas branch. Everywhere except near the specks, both paths must agree byte-for-byte.

**Disagreement threshold in path rendering**: Pixels with coverage difference >24 (out of 255) are counted as disagreement. This tolerates antialiasing/rasterization variance while catching genuine geometric differences. The threshold is chosen to be large enough that rasterizer differences don't matter but small enough to catch real errors.

### Unclear intent

None identified. All test purposes are explicit in documentation comments or directly observable from assertions.
