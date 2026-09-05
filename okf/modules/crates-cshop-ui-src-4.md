---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-ui/src (chrome)'
files:
- crates/cshop-ui/src/chrome.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-ui/src (chrome)
type: Module
---

### What it does

Implements the application chrome UI for C-Shop: the custom title bar with window controls and menus, the toolbox with tool selection and color swatches, the tool options bar, and window resize borders. Handles window operations (move, resize, maximize, minimize, close) and presents the main menu structure for file, edit, image, layer, selection, filter, and view operations.

### Public interface

```rust
pub const TITLE_BAR_HEIGHT: f32 = 30.0;

pub fn title_bar(app: &mut CShopApp, ui: &mut egui::Ui)
pub fn resize_borders(app: &mut CShopApp, ui: &mut egui::Ui)
pub fn menu_bar(app: &mut CShopApp, ui: &mut egui::Ui) -> f32
pub fn toolbox(app: &mut CShopApp, ui: &mut egui::Ui)
pub fn foreground_swatch_id() -> egui::Id
pub fn background_swatch_id() -> egui::Id
pub fn options_bar(app: &mut CShopApp, ui: &mut egui::Ui)
```

### Key invariants

- The drag handle for moving the window must be registered before interactive elements (menus, buttons) so that clicks on those elements take priority over window dragging.
- Resize zones overlap at corners; corners are registered before edge zones so they win the overlap.
- A maximized window has no resize borders.
- The title bar's free-to-drag area excludes the menus and window buttons; only empty space initiates a window move.
- The toolbox displays the currently selected tool from each group, remembering the user's last choice within that group.
- Multi-tool slots show a corner dot indicator and support cycling via repeated clicks or context menu selection.
- Tool options only appear when a document exists, or are replaced by transform options during an active transform.

### Non-obvious decisions

- The title bar's drag handle is registered *first* before menus and buttons are drawn, despite being acted upon last. This is deliberate: egui renders widgets added later on top, so registering the drag handle first ensures menus and buttons can capture clicks and prevent the window from moving when interacting with them.
- Window icon glyphs (minimize, maximize, restore, close) are drawn procedurally rather than using font glyphs, because these symbols are not in egui's bundled fonts.
- The `menu_bar` function returns the x-coordinate of its end rather than using `MenuBar`'s response, because `MenuBar` fills its allocated width and the title bar needs to know where menus actually end to determine the free drag area.
- Foreground and background swatch IDs are fixed rather than derived from the enclosing `Ui`, so tests can query egui for their actual screen positions without recomputing layout.
- The background color swatch is registered as interactive before the foreground swatch, despite being drawn behind it, so that the foreground can win the overlap when both are clicked.

### Unclear intent

The `options_bar` function is incomplete in the provided source (ends at `percent_slider(&mut app.gradient.opacity)` without closing braces), so the full extent of tool-specific options cannot be determined from this excerpt alone.
