---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-core/src (history)'
files:
- crates/cshop-core/src/history.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-core/src (history)
type: Module
---

### What it does

Implements reversible commands and a bounded undo/redo stack for C-Shop's document editing. Each command knows how to apply and revert itself, with specialized variants for pixel operations, layer transformations, and metadata changes. The history maintains a memory budget to prevent unbounded growth on large documents.

### Public interface

```rust
pub trait Command: std::fmt::Debug + std::any::Any + Send {
    fn name(&self) -> String;
    fn apply(&mut self, doc: &mut Document) -> Dirty;
    fn revert(&mut self, doc: &mut Document) -> Dirty;
    fn merge(&mut self, _next: &dyn Command) -> bool { false }
    fn memory_bytes(&self) -> u64 { 0 }
}

pub struct History {
    pub fn new(origin: impl Into<String>) -> Self;
    pub fn with_limit(mut self, limit: usize) -> Self;
    pub fn with_budget(mut self, bytes: u64) -> Self;
    pub fn memory_bytes(&self) -> u64;
    pub fn forgotten(&self) -> usize;
    pub fn origin(&self) -> &str;
    pub fn cursor(&self) -> usize;
    pub fn can_undo(&self) -> bool;
    pub fn can_redo(&self) -> bool;
    pub fn undo_name(&self) -> Option<String>;
    pub fn redo_name(&self) -> Option<String>;
    pub fn labels(&self) -> Vec<String>;
    pub fn apply(&mut self, doc: &mut Document, cmd: Box<dyn Command>) -> Dirty;
    pub fn undo(&mut self, doc: &mut Document) -> Option<Dirty>;
    pub fn redo(&mut self, doc: &mut Document) -> Option<Dirty>;
    pub fn jump_to(&mut self, doc: &mut Document, target: usize) -> Dirty;
}

pub const DEFAULT_MEMORY_BUDGET: u64 = 2 << 30;

// Command implementations:
pub struct AddLayer { ... }
pub struct Compound { ... }
pub struct DeleteLayer { ... }
pub struct MoveLayer { ... }
pub struct OffsetLayer { ... }
pub struct SetLayerProperty { ... }
pub struct ReplacePixels { ... }
pub struct ReplaceLayerPixels { ... }
pub struct ResizeCanvas { ... }
pub struct ResizeImage { ... }
pub struct RasterizeLayer { ... }
pub struct SetLayerEffects { ... }
pub struct SetShapeContent { ... }
pub struct SetTextContent { ... }
```

### Key invariants

- Commands must be idempotent with respect to stored state: `apply()` may be called multiple times (during redo), so implementations capture needed state on first application and reuse it.
- The history cursor always points to the count of applied entries; entries at or past this index are redoable.
- At least one entry is always preserved when trimming, even if it exceeds the memory budget alone, to guarantee some undo exists.
- Raster operations store only the rectangular region they touch in `before` form, not full-layer copies, to bound memory usage on large documents.
- Commands that merge (like slider drags) keep the original `from` state so undo jumps to the start of the gesture, not each intermediate step.

### Non-obvious decisions

- **Uniform region compression**: The `Stored` enum distinguishes between uniform-colour regions (stored as width/height/color) and photographic content (stored as raw pixels). The uniform check uses `par_iter().all()` which short-circuits on the first differing pixel, making it fast for typical edits while still accurately detecting truly uniform regions. This avoids storing 275 MB per full-canvas fill on large documents.

- **Trimming on apply rather than lazy eviction**: When a new edit invalidates the redo tail, trimming happens immediately during `apply()`, not deferred. This ensures the cursor position remains valid without additional logic, since removed entries shift it down proportionally.

- **Compound commands for multi-step gestures**: Operations like "Copy Layer" that must atomically add a layer and clear the selection are wrapped in `Compound` so they undo as a single step. This prevents the intermediate state (new layer, no selection) from being visible if the user undo once.

- **Layer snapshots for ResizeImage**: Unlike canvas resize (which only shifts layers), image resize snapshots all raster layers and masks before resampling. This is expensive but justified because image resize is rare and deliberate; an exact undo is worth the memory cost.

- **Memory budget over entry count**: The history is bounded by both a max entry count (default 200) and total memory budget (default 2 GB), with trimming respecting both. A count alone cannot work because a brush stroke on a 6000×6000 canvas can be kilobytes while a full-canvas fill is 275 MB.

### Unclear intent

- The `Dirty` struct and its `merge` method are imported from the document module but not defined here. The exact semantics of region vs. structural vs. pixel-level dirtiness would benefit from cross-reference to that module.
