---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-ui/src (app)'
files:
- crates/cshop-ui/src/app.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-ui/src (app)
type: Module
---

### What it does

The main application state and frame update loop for C-Shop, a GPU-accelerated image editor. It manages documents, tools, painting strokes, dialogs, and coordinates rendering across the UI, canvas, and GPU compositor.

### Public interface

```rust
pub struct CShopApp {
    pub gpu: GpuContext,
    pub docs: Vec<DocView>,
    pub active: Option<usize>,
    pub tool: Tool,
    pub foreground: Rgba8,
    pub background: Rgba8,
    pub brush: Brush,
    pub text_style: cshop_core::text::TextStyle,
    pub text_edit: Option<crate::text_tool::TextEdit>,
    pub clipboard: crate::clipboard::Clipboard,
    pub shape_kind: cshop_core::shape::ShapeKind,
    pub shape_style: cshop_core::shape::ShapeStyle,
    pub drag_start: Option<Vec2>,
    pub pen: Option<PenDraft>,
    pub now: f64,
    pub selection_mode: SelectionMode,
    pub selection_feather: f32,
    pub selection_antialias: bool,
    pub wand: WandOptions,
    pub sample_all_layers: bool,
    pub quick_mask: bool,
    pub bucket: cshop_core::fill::BucketOptions,
    pub gradient: cshop_core::fill::Gradient,
    pub gradient_drag: Option<(Vec2, Vec2)>,
    pub clone_anchor: Option<Vec2>,
    pub clone_aligned: bool,
    pub dialog: Dialog,
    pub toast: Option<(String, bool)>,
    pub show_panels: bool,
    pub canvas_viewport: egui::Rect,
    pub drag: Option<SelectionDrag>,
    pub transform: Option<ActiveTransform>,
    pub crop: Option<ActiveCrop>,
    pub last_filter: Option<cshop_core::filters::Filter>,
    pub window_commands: Vec<WindowCommand>,
    pub is_maximized: bool,
    pub quit: bool,
}

impl CShopApp {
    pub fn new(gpu: GpuContext) -> Self
    pub fn doc(&self) -> Option<&DocView>
    pub fn doc_mut(&mut self) -> Option<&mut DocView>
    pub fn push(&mut self, action: Action)
    pub fn window(&mut self, command: WindowCommand)
    pub fn logo(&mut self, ctx: &egui::Context) -> egui::TextureHandle
    pub fn histogram(&mut self) -> Option<&crate::properties::Histogram>
    pub fn update(&mut self, ui: &mut egui::Ui, renderer: &mut egui_wgpu::Renderer)
    pub fn text_layer_at(&self, at: Vec2) -> Option<LayerId>
    pub fn editing_text_contains(&self, at: Vec2) -> bool
    pub fn text_caret_rect(&self) -> Option<(Vec2, Vec2)>
}
```

### Key invariants

- `active` is always `< docs.len()` when `Some`, or `None` when `docs` is empty.
- Only the active document is composited each frame; inactive documents must be invalidated when they become active.
- A stroke in progress (`self.stroke`) holds a snapshot of the target buffer as it was when the stroke began, allowing painting to re-derive rather than compound, ensuring preview matches committed result.
- The histogram is keyed on `(document_id, history_cursor)` and only recomputed when the document changes, not during interactive slider drags.
- The Layer Style dialog remains movable with a fixed window ID so its position persists across frames and re-openings.
- Text editing has priority over tool shortcuts; the keyboard only belongs to the editor when `text_edit.is_some()`.

### Non-obvious decisions

- **Painting re-derives from snapshots each frame** rather than compounding onto the live buffer. This ensures the preview exactly matches the committed result and makes undo entries precise without needing to store intermediate states.
- **Histogram is gated on history cursor position** rather than recomputed every frame. GPU read-backs are expensive; keying on the history cursor lets sliders drag without stalling, only refreshing when the image actually changes.
- **Logo is embedded and decoded on first use** rather than loaded from disk. A built executable carries no external asset dependencies and cannot lose the logo file.
- **The Layer Style dialog is a movable window, not modal** while other dialogs are modal. It needs to sit beside the canvas (its preview) rather than occlude it, making occlusion undesirable.
- **Tool selection commits text editing** unconditionally. Switching tools while editing implicitly finalizes the text layer, preventing confusion about which tool is active versus which one is drawing.
- **Action queue drains up to 8 iterations** with a safety limit. Some actions can chain (e.g., Save → Save As), so the loop settles cascading commands without unbounded recursion.

### Unclear intent

- `shape_synced: Option<LayerId>` tracks which shape layer the options bar last loaded from, but the mechanism for "adopting its settings instead of overwriting them" is not visible in this file and depends on shape tool implementation elsewhere.
