---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-core/src (document)'
files:
- crates/cshop-core/src/document.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-core/src (document)
type: Module
---

### What it does

Defines the document model: a canvas with metadata (dimensions, DPI, name) and a tree of layers that can be edited. Tracks the active layer, pixel/mask selections, and what changed ("dirty" state) so the renderer knows what to recompose. Owns no undo history; that lives separately and mutates this document through commands.

### Public interface

```rust
pub struct DocumentId(pub u64)
impl DocumentId {
    fn fresh() -> Self
}

pub struct Dirty {
    pub layers: Vec<LayerId>,
    pub rect: IRect,
    pub structure: bool,
}
impl Dirty {
    pub const NONE: Dirty
    pub fn region(rect: IRect) -> Self
    pub fn pixels(layer: LayerId, rect: IRect) -> Self
    pub fn structural(rect: IRect) -> Self
    pub fn is_empty(&self) -> bool
    pub fn merge(&mut self, other: Dirty)
}

pub struct AlphaChannel {
    pub name: String,
    pub data: MaskBuffer,
    pub visible: bool,
}

pub enum EditTarget {
    Pixels,
    Mask,
}

pub enum Background {
    White,
    Transparent,
    Color(Rgba8),
}

pub struct Document {
    pub id: DocumentId,
    pub name: String,
    pub width: u32,
    pub height: u32,
    pub dpi: f32,
    pub tree: LayerTree,
    pub active: Option<LayerId>,
    pub selected_layers: Vec<LayerId>,
    pub selection: Option<Selection>,
    pub path: Option<PathBuf>,
    pub modified: bool,
    pub channels: Vec<AlphaChannel>,
    pub edit_target: EditTarget,
    pub last_selection: Option<Selection>,
}
impl Document {
    pub fn new(name: impl Into<String>, width: u32, height: u32, background: Background) -> Self
    pub fn from_image(name: impl Into<String>, pixels: PixelBuffer) -> Self
    pub fn bounds(&self) -> IRect
    pub fn editable_bounds(&self) -> IRect
    pub fn is_editable(&self, x: i32, y: i32) -> bool
    pub fn selection_coverage(&self, x: i32, y: i32) -> f32
    pub fn set_selection(&mut self, selection: Option<Selection>)
    pub fn active_has_mask(&self) -> bool
    pub fn effective_edit_target(&self) -> EditTarget
    pub fn add_channel(&mut self, data: MaskBuffer) -> usize
    pub fn has_selection(&self) -> bool
    pub fn active_layer(&self) -> Option<&Layer>
    pub fn active_layer_mut(&mut self) -> Option<&mut Layer>
    pub fn select(&mut self, id: Option<LayerId>)
    pub fn select_add(&mut self, id: LayerId)
    pub fn prune_selection(&mut self)
    pub fn next_layer_name(&self) -> String
    pub fn content_bounds(&self) -> IRect
    pub fn memory_bytes(&self) -> u64
}
```

### Key invariants

- A document always has a fresh `DocumentId` when created or cloned, even if the content is identical; layer IDs restart at 1 per document, so caches keyed by layer ID need document identity to avoid collisions.
- `selected_layers` always contains `active` when `active` is `Some`; they are paired.
- An empty selection (`Selection::is_empty()`) is stored as `None` rather than `Some(empty)`; otherwise deselecting would silently lock down the whole canvas.
- After any structural change to the tree (add/remove/reorder), `prune_selection()` must be called to drop dangling layer IDs and ensure `active` points to an existing layer.
- A non-empty document always ends up with at least one selected layer after pruning; an empty document has `active = None`.
- `effective_edit_target()` returns `Mask` only if both `edit_target == Mask` *and* the active layer has a mask; otherwise it falls back to `Pixels`.

### Non-obvious decisions

- **Separation of selected layers and pixel selection**: `selected_layers` is the multi-selection for bulk layer operations, while `selection` is the pixel/alpha selection for painting constraints. They are orthogonal concepts called by different names ("selected layers" vs. "selection") to avoid confusion.
- **`DocumentId` generation on clone**: Cloning a document mints a new ID rather than copying the original. This forces cache invalidation and prevents layer ID collisions in per-layer caches (notably GPU texture caches) when a document is duplicated.
- **Storing `last_selection` on deselect**: When a selection is cleared, it is preserved so `Reselect` can restore it. This keeps the undo interface clean — deselection itself can be undone by remembering what was selected.
- **Zero-sized documents clamped to 1×1**: Dimensions are clamped to a minimum of 1 in constructors rather than allowing 0, avoiding degenerate edge cases in rendering and layer bounds calculations.
