---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-app/src (window)'
files:
- crates/cshop-app/src/window.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-app/src (window)
type: Module
---

### What it does

Manages the application window, GPU surface, and event loop for C-Shop. Coordinates between winit (window events), egui (UI framework), and wgpu (GPU rendering) to display and update the interface, while handling window operations like resize, drag, and maximize.

### Public interface

```rust
pub fn run(files: Vec<String>, started: Instant) -> Result<(), Box<dyn std::error::Error>>
```

Entry point that creates the event loop and begins processing window events. Accepts file paths to open on startup and a timestamp marking process start.

### Key invariants

- The `State` is only created once, in the `resumed` handler, and persists until the application exits.
- Surface reconfiguration follows every resize and occurs before attempting to acquire a new frame.
- egui input is consumed before the window command queue is run, ensuring UI interactions are processed in the correct order.
- Window requests for redraw are only issued when egui signals `repaint_delay.is_zero()` or when explicitly triggered by resize/focus events, maintaining the `ControlFlow::Wait` model.
- The surface format is always sRGB to match egui and the present pass expectations.

### Non-obvious decisions

The window is created with `decorations(false)` and a custom title bar is drawn by the UI layer (`chrome::title_bar`). This allows the application to style the entire chrome uniformly rather than adapting to platform conventions.

Window commands (drag, resize, minimize, etc.) are executed *after* the egui frame is rendered and platform output is handled. This ensures the drag begins from a "settled" state and prevents input events from being processed mid-operation.

On `Acquired::Suboptimal`, the frame is still rendered after reconfiguring the surface. On `Acquired::Outdated` or `Acquired::Lost`, the frame is skipped entirely. This distinction prevents unnecessary work when the swapchain is merely suboptimal while still recovering from hard failures.

The `reported_startup` flag logs startup time only after the first frame is presented, not after initialization completes. This captures the full time to first visible output rather than just setup time.
