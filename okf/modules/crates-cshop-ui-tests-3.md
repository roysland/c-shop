---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-ui/tests (shortcuts, text,
  tools)'
files:
- crates/cshop-ui/tests/shortcuts.rs
- crates/cshop-ui/tests/text.rs
- crates/cshop-ui/tests/tools.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-ui/tests (shortcuts, text, tools)
type: Module
---

### What it does

Integration tests for the C-Shop UI layer, covering keyboard shortcuts, text tool functionality, and painting tools (bucket fill, gradient, clone stamp). Tests verify that user interactions through the interface produce correct effects on documents, using a test harness that simulates real input.

### Public interface

```rust
// From shortcuts.rs
fn ready() -> Option<Harness>
fn merge_down_and_quit_are_bound_not_just_advertised()
fn the_backspace_family_fills_the_conventional_way()
fn shift_backspace_variants_preserve_transparency()
fn the_adjustment_chords_open_their_dialogs()
fn ctrl_alt_i_resizes_without_also_inverting()
fn adding_shift_picks_the_other_command_not_both()
fn the_brackets_size_the_brush_and_step_back_exactly()
fn a_digit_sets_the_painting_opacity()
fn ctrl_bracket_restacks_the_active_layer()
fn ctrl_j_duplicates_and_ctrl_alt_g_clips()
fn ctrl_j_copies_only_the_selection()
fn a_feathered_selection_copies_a_soft_edge()
fn undoing_layer_via_copy_restores_the_selection_too()

// From text.rs
fn ready() -> Option<Harness>
fn type_text(h: &mut Harness, s: &str)
fn active_text(h: &Harness) -> Option<String>
fn clicking_starts_a_type_layer_and_typing_fills_it()
fn a_type_layer_actually_paints_pixels()
fn committing_empty_type_leaves_nothing_behind()
fn escape_abandons_a_new_layer_and_restores_an_edited_one()
fn a_whole_editing_session_is_one_undo_step()
fn the_caret_moves_and_edits_where_it_is()
fn enter_breaks_the_line_and_up_down_cross_it()
fn the_caret_has_a_position_to_draw()
fn dragging_makes_a_paragraph_box_that_wraps()
fn type_cannot_be_painted_on_until_it_is_rasterised()
fn clicking_existing_type_reopens_it()
fn type_behaves_like_any_other_layer()
fn moving_type_then_editing_it_keeps_it_where_it_was_put()
fn right_aligned_type_grows_away_from_its_anchor()
fn clicking_the_canvas_with_the_type_tool_starts_type()

// From tools.rs
fn app_with(w: u32, h: u32) -> Option<CShopApp>
fn set_pixels(app: &mut CShopApp, px: PixelBuffer)
fn pixels(app: &CShopApp) -> &PixelBuffer
// Paint Bucket tests
fn the_bucket_fills_the_region_under_the_click()
fn the_bucket_respects_the_selection()
fn the_bucket_honours_tolerance()
fn a_locked_layer_refuses_the_bucket()
// Gradient tests
fn a_gradient_drag_lays_down_a_ramp()
fn a_gradient_respects_the_selection()
fn every_gradient_type_draws_something()
fn a_gradient_drag_of_no_length_does_nothing()
// Clone Stamp tests
fn the_clone_stamp_copies_from_its_anchor()
fn the_clone_stamp_needs_an_anchor_first()
fn the_clone_stamp_keeps_every_brush_control()
fn an_unaligned_clone_restarts_from_the_anchor()
fn the_clone_stamp_works_at_every_hardness()
fn an_aligned_clone_whose_source_leaves_the_image_says_so()
// Fill and color picker tests
fn fill_with_lays_down_the_chosen_colour()
fn fill_opacity_and_mode_are_applied()
fn fill_can_preserve_transparency()
fn the_colour_picker_sets_the_chosen_swatch()
fn the_new_tools_do_not_panic_without_a_document()
```

### Key invariants

- **Keyboard modifier exactness**: Chord matching must distinguish between `Ctrl+I`, `Ctrl+Alt+I`, `Ctrl+D`, and `Ctrl+Shift+D` to prevent unintended command cascades. A superfluous match (e.g., `Ctrl+D` firing during `Ctrl+Shift+D` processing) would corrupt document state.
- **Undo granularity**: Complex operations like "Layer via Copy" must restore both the created layer and the original selection in a single undo step. Keystroke-level undo would leave documents in inconsistent states.
- **Selection preservation during fill**: Fill operations must respect layer transparency locks and selection boundaries. Transparency-preserving fills must not paint where layers are already transparent.
- **Text layer anchor invariance**: When a text layer is moved then reopened for editing, the anchor position must move with it. Editing must not teleport the rendered text back to its original creation point.
- **Right-aligned text growth**: Right-aligned point text expands leftward during typing; the right edge must remain fixed while the layer's raster corner moves and the anchor stays constant.
- **Clone stamp source validity**: Aligned cloning must validate that the source pixel coordinate remains within the image bounds. Out-of-bounds sources produce no output and no undo entry.
- **Tool operation without documents**: Bucket fill, gradient, clone stamp, and dialogs must not panic when called with no open document; they simply do nothing.

### Non-obvious decisions

- **Headless GPU context fallback**: `tools.rs` skips tests with `GpuContext::headless()` errors rather than failing. GPU support is optional in test environments (CI without display/drivers), so graceful skipping is preferred to test failure.
- **Empty text layer cleanup**: Committing a text layer without any typed content removes the layer entirely rather than leaving an empty layer. This prevents clutter and distinguishes between "user started typing but changed their mind" and "user created intentional empty layers."
- **Feathered selection via Copy**: `a_feathered_selection_copies_a_soft_edge()` verifies that feathering produces soft edges (partial alpha) rather than hard cut-outs. The selection's feather blur must propagate into the rasterized layer's alpha channel.
- **Tolerance as integer diff**: Bucket fill tolerance is tested with delta differences (e.g., 20 levels apart with tolerance 5 should not fill, but tolerance 40 should). This is testing the actual comparison logic rather than asserting a specific algorithm.

### Unclear intent

- **`h.settle(2)` and `h.settle(3)` call counts**: The specific numbers (2 or 3) for harness settlement after document creation or text operations appear arbitrary. It is unclear whether these represent frame counts, render passes, or timing thresholds, or whether the exact values matter or are conservative upper bounds. Refer to `crates/cshop-ui/src-*` for harness implementation.
