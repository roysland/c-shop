---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-ui/src (dialogs, doc_view)'
files:
- crates/cshop-ui/src/dialogs.rs
- crates/cshop-ui/src/doc_view.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-ui/src (dialogs, doc_view)
type: Module
---

### What it does

Provides modal dialogs for user input (new documents, file browser, sizing, filling, color picking, layer renaming) and maintains per-document GPU state including composited textures, view transform, and layer thumbnails.

### Public interface

**dialogs.rs:**
- `Dialog` enum: variants `None`, `NewDocument`, `FileBrowser`, `Modify`, `ImageSize`, `Filter`, `Rename`, `Adjustment`, `LayerStyle`, `Fill`, `ColorPicker`, `About`
- `Dialog::is_open() -> bool`
- `NewDocument::new() -> Self`; `.background() -> Background`; `.ui(&mut self, ui: &mut egui::Ui, actions: &mut Vec<Action>) -> bool`
- `FillDialog::new(foreground: Rgba8, background: Rgba8) -> Self`; `.color() -> Rgba8`; `.ui(&mut self, ui: &mut egui::Ui, actions: &mut Vec<Action>) -> bool`
- `ColorPickerDialog::new(target: PickerTarget, current: Rgba8) -> Self`; `.ui(&mut self, ui: &mut egui::Ui, actions: &mut Vec<Action>) -> bool`
- `RenameDialog::new(layer: LayerId, name: String) -> Self`; `.ui(&mut self, ui: &mut egui::Ui, actions: &mut Vec<Action>) -> bool`
- `SizeDialog::image(width: u32, height: u32) -> Self`; `.canvas(...) -> Self`; `.ui(&mut self, ui: &mut egui::Ui, actions: &mut Vec<Action>) -> bool`
- `ModifyDialog::new(kind: ModifyKind) -> Self`; `.ui(&mut self, ui: &mut egui::Ui, actions: &mut Vec<Action>) -> bool`
- `FileBrowser::new(mode: BrowserMode, start: Option<PathBuf>) -> Self`; `.ui(&mut self, ui: &mut egui::Ui, actions: &mut Vec<Action>) -> bool`

**doc_view.rs:**
- `DocView::new(gpu: &GpuContext, doc: Document, origin_label: &str) -> Self`
- `.pending_rect() -> IRect`
- `.mark_dirty(&mut self, dirty: Dirty)`
- `.invalidate(&mut self)`
- `.resize_targets(&mut self, gpu: &GpuContext)`
- `.cache() -> &LayerTextures`
- `.sync(&mut self, gpu: &GpuContext, compositor: &mut Compositor, renderer: &mut egui_wgpu::Renderer)`
- Public fields: `doc: Document`, `history: History`, `center: egui::Vec2`, `zoom: f32`, `zoom_initialised: bool`
- Constants: `ZOOM_STOPS: &[f32]`, `MIN_ZOOM: f32 = 0.0016`, `MAX_ZOOM: f32 = 32.0`

### Key invariants

**dialogs.rs:**
- Dialog `ui()` methods return `true` when the dialog should close; they push `Action`s to the provided vector for execution.
- `NewDocument` presents presets that users reach for; preset selections overwrite width/height/dpi atomically.
- `FileBrowser` filters files by extension in Open mode; Save mode allows any filename.
- `SizeDialog` enforces aspect ratio constraints when `link_aspect` is true.

**doc_view.rs:**
- `DocView` owns a `LayerTextures` cache scoped to its document, preventing layer-id collisions across tabs.
- `pending` tracks dirty regions; `needs_full` forces complete recomposition on structural changes.
- Edits reaching outside a layer's own pixels (via effects like blur) expand `dirty.rect` by `padding()` to prevent stale composites.
- `epoch` counter invalidates cached thumbnails when their source layer changes; structural changes clear all thumbnails.
- Zoom level determines GPU texture filter: `Nearest` above 100%, `Linear` below (for hard pixel edges vs blur).

### Non-obvious decisions

**dialogs.rs:**
- Filter and Adjustment dialogs are boxed in the `Dialog` enum because they carry preview images and histograms that would inflate the size of every `Dialog::None` variant significantly.
- File browser filters hidden files without providing a UI toggle, assuming users never want them in a graphical file picker.
- `FileBrowser::target()` auto-appends the format's default extension if the user-typed filename doesn't already have one matching the selected format, preventing accidental mismatches in Save mode.

**doc_view.rs:**
- Layer effect padding inflates the recomposite region lazily (queried per `mark_dirty` call) rather than baking it into layer metadata, keeping effect logic localized to `cshop_core::effects`.
- Thumbnails are keyed by `LayerId` with a high bit (`MASK_THUMB_BIT`) set for mask thumbnails, reusing a single cache rather than maintaining separate structures; this works because layer IDs are small counters that never use the high bits.
- Filter mode switches at exactly `zoom >= 1.0` rather than at 1.5 or another threshold, prioritizing the common case: pixel-aligned viewing above 100% zoom.

### Unclear intent

**doc_view.rs:**
- The `.sync()` method's signature is incomplete in the provided source (cut off mid-function). Its full logic and return value cannot be determined.
