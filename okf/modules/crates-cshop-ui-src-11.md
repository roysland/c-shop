---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-ui/src (tools, transform_tool)'
files:
- crates/cshop-ui/src/tools.rs
- crates/cshop-ui/src/transform_tool.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-ui/src (tools, transform_tool)
type: Module
---

### What it does

`tools.rs` defines the complete toolbox available to users, organized into groups that share keyboard shortcuts, with methods for name lookup and enumeration. `transform_tool.rs` implements Free Transform and Crop operations on layers, allowing quadrilateral corner manipulation, rotation, scaling, and perspective distortion through projective matrices.

### Public interface

**tools.rs**
```rust
pub enum Tool { Move, RectangularMarquee, /* ... */, Zoom }

impl Tool {
    pub fn name(self) -> &'static str
    pub fn glyph(self) -> &'static str
    pub fn from_name(name: &str) -> Option<Tool>
    pub fn is_implemented(self) -> bool
    pub fn uses_brush(self) -> bool
    pub fn is_selection_tool(self) -> bool
}

pub struct ToolGroup {
    pub key: egui::Key
    pub label: char
    pub tools: &'static [Tool]
}

pub const TOOL_GROUPS: &[ToolGroup]
pub fn group_of(tool: Tool) -> Option<&'static ToolGroup>
pub fn cycle(group: &ToolGroup, current: Tool) -> Tool
```

**transform_tool.rs**
```rust
pub const HANDLE_GRAB: f32
const PROXY_MAX: u32

pub struct ActiveTransform {
    pub layer: LayerId
    pub source: PixelBuffer
    pub source_offset: (i32, i32)
    pub source_mask: Option<LayerMask>
    pub source_rect: IRect
    pub corners: [Vec2; 4]
    pub dragging: Option<Handle>
    pub filter: Resampling
    pub proxy: PixelBuffer
    pub modified: bool
}

impl ActiveTransform {
    pub fn begin(layer: LayerId, source: PixelBuffer, source_offset: (i32, i32), source_mask: Option<LayerMask>) -> Self
    pub fn centre(&self) -> Vec2
    pub fn handle_position(&self, handle: Handle) -> Vec2
    pub fn hit(&self, point: Vec2, zoom: f32) -> Option<Handle>
    pub fn contains(&self, p: Vec2) -> bool
    pub fn begin_drag(&mut self, handle: Handle, at: Vec2)
    pub fn drag_to(&mut self, at: Vec2, distort: bool, constrain: bool, from_centre: bool)
    pub fn end_drag(&mut self)
    pub fn matrix(&self) -> Option<Transform>
    pub fn render(&self, clip: IRect) -> Option<(PixelBuffer, (i32, i32))>
    pub fn reset(&mut self)
    pub fn scale_percent(&self) -> (f32, f32)
    pub fn rotation_degrees(&self) -> f32
}

pub struct ActiveCrop {
    pub rect: IRect
    pub dragging: Option<Handle>
    pub aspect: Option<f32>
}

impl ActiveCrop {
    pub fn new(rect: IRect) -> Self
    pub fn handle_position(&self, handle: Handle) -> Vec2
    pub fn hit(&self, point: Vec2, zoom: f32) -> Option<Handle>
    pub fn begin_drag(&mut self, handle: Handle, at: Vec2)
    pub fn drag_to(&mut self, at: Vec2, bounds: IRect)
    pub fn end_drag(&mut self)
}
```

### Key invariants

**tools.rs**
- Every tool belongs to exactly one `ToolGroup` (enforced by test).
- Tool shortcuts are unique across all groups.
- All tools in the enum appear in `TOOL_GROUPS`.
- `is_implemented()` is maintained explicitly; adding a tool shows as unimplemented until added to the match.

**transform_tool.rs**
- `ActiveTransform.corners` always contains exactly four points defining the current quadrilateral, starting top-left and proceeding clockwise.
- `start_corners` is only valid during an active drag and captures the state at `begin_drag`.
- The `proxy` is built once at initialization and never resized; it's only used for preview rendering during drag.
- `ActiveCrop.rect` must always be non-empty or very close to it (minimum 1×1 after intersection with bounds).
- When a crop edge is dragged past its opposite, the rectangle is flipped (coordinates swapped) rather than inverted.
- `modified` is only set to `true` once an actual drag movement occurs.

### Non-obvious decisions

**transform_tool.rs**
- **Live preview proxy**: The layer is resampled once at `begin()` into a downscaled copy (`proxy`) for frame-by-frame preview during drag, with the real transformation applied only at `render()` on commit. This avoids stalling interactive drag on high-resolution layers.
- **Projective matrix model**: Free Transform uses a single 2D → 2D matrix derived from source and destination quads rather than separate scale/rotate/skew modes. This means any handle movement is always meaningful and corner distortion, perspective, and rigid transformations all fall out of one computation path.
- **Opposite corner as anchor**: When dragging a non-corner handle without Alt, the scale pivots on the opposite handle rather than the center, matching conventional image editor behavior and allowing intuitive one-handed resizing.
- **Safe division in scale**: The `safe_ratio()` helper guards against zero-width intermediate states during drag-to-anchor moves by returning 1.0, preventing division by zero and numerical explosion.

### Unclear intent

**tools.rs**
- `Tool::from_name()` accepts both exact names (case-insensitive, punctuation-stripped) and unambiguous prefixes. The design is clear, but the use case — `--demo-tool` screenshot flag mentioned in the comment — is not documented here and would require checking the caller.
