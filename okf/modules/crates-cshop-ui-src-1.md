---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-ui/src (adjust_ui)'
files:
- crates/cshop-ui/src/adjust_ui.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-ui/src (adjust_ui)
type: Module
---

### What it does

Implements a modal dialog for image adjustments (Curves, Levels, etc.) that allows users to preview changes before applying them. The dialog maintains a downscaled proxy of the affected region and re-renders it whenever settings change, only touching the real image pixels when the user clicks OK.

### Public interface

```rust
impl AdjustmentDialog {
    pub fn new(adjustment: Adjustment, source: &PixelBuffer) -> Self
    pub fn title(&self) -> String
    pub fn ui(&mut self, ui: &mut egui::Ui, actions: &mut Vec<Action>) -> bool
}
```

### Key invariants

- The preview proxy is never larger than 320 pixels on its longest edge, reducing computational cost while remaining useful for preview purposes.
- The histogram is always computed from the full-resolution source, not the downscaled proxy, to preserve peak information needed for Levels adjustments.
- The preview texture is only re-rendered when the adjustment settings change from the last render; unchanged settings reuse the cached preview.
- The original source pixels are never modified until the user clicks OK; all adjustments are applied only to the proxy for preview.

### Non-obvious decisions

The histogram is deliberately computed from the full-resolution source rather than the proxy. The code comment explains this: downscaling averages away the very peaks that a Levels adjustment typically targets, so using the proxy histogram would give misleading feedback.

The preview texture is updated in-place rather than allocated fresh on each change. This is an optimization for responsiveness during interactive adjustments like dragging curve points, which would otherwise cause texture allocation churn.
