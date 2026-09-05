# Modular rewrite plan: a basic plugin-based image editor

A second pass over [`fork-assessment.md`](fork-assessment.md). The first pass
answered *whether* the code could carry a modular system. This pass answers
*how* to turn it into one, scoped to the stated goal: **a basic modular image
editor**, not a full Photoshop clone. Written against the same tree
(`cargo check --workspace --tests` passes; core/io tests pass; UI tests need a
real GPU).

## What "modular" should mean here

The original creator's stated intent — move away from a monolith — is sound,
but the assessment found the execution went the opposite way: **closed enums
everywhere and one 3,800-line god object.** Adding anything touches ~8 compile
sites and grows a 660-line `match`.

For a *fork* whose goal is to split work per-feature, "modular" does **not**
mean dynamically-loaded `.so`/`.dll` plugins. That path is a poor fit for this
codebase (pure-Rust constraint, `panic = "abort"`, no ABI story) and overkill
for a basic editor. It means:

> **In-process plugins: self-contained Rust modules that register themselves
> into a small set of runtime registries, so a feature is one file/one folder
> you can build, test, debug and enable in isolation — never an edit spread
> across eight files.**

This keeps Rust's compile-time safety (the thing the assessment praised) while
removing the thing it criticised (edits smeared across the codebase). Each
plugin becomes a unit of work a fork contributor can own.

## The core idea: three registries + a context

Replace the three closed extension points with three trait-object registries,
each populated at startup. Everything else stays as-is.

| Registry | Replaces | A plugin provides |
|---|---|---|
| `CommandRegistry` | menu bar / shortcuts / context-menu triplication | one `Command` descriptor: id, label, menu path, shortcut, `enabled`, `run` |
| `ToolRegistry` | the `Tool` enum + 8 touch points | one `Tool` trait impl owning its own state + pointer handlers |
| `EffectRegistry` | *(already good — generalise it)* | one `Operation` (filter/adjustment) with params + `apply` |

Each registry hands plugins a single `EditorCtx` handle — the mediated seam
into the document, selection, history and colour state — so a plugin never
touches `CShopApp`'s 60 fields directly. That handle **is** the modular
boundary; get it right and the god object dissolves naturally.

```
                 ┌─────────────────────────────────────────┐
   plugins  ───▶ │  registries (Command / Tool / Effect)    │
                 └───────────────────┬─────────────────────┘
                                     │  &mut EditorCtx
                 ┌───────────────────▼─────────────────────┐
                 │  EditorShell (old CShopApp, now thin)     │
                 │  owns docs, active index, registries      │
                 └───────────────────┬─────────────────────┘
                                     │
                 cshop-core (document, history) — unchanged
```

## What stays exactly as it is

Per the instruction ("existing features can stay if it's easier"), keep:

- **`cshop-core` entirely.** It is pure, tested, and already the right shape.
  The filter/adjustment enums there are *data catalogs*, not the problem — the
  problem is how the UI consumes them. No core changes needed for phase 1–3.
- **`cshop-gpu`, `cshop-io`.** Untouched. Not extension surfaces for a basic
  editor.
- **The `Action` enum and single `run()` seam.** This is the codebase's best
  feature. We do **not** delete it. The `CommandRegistry` *emits* `Action`s;
  plugins that need bespoke mutation get a scoped `EditorCtx` instead of a new
  `Action` variant. The enum stops growing without being torn out.
- **The `.style` system, script harness, MCP server.** All already "open
  doors". Left alone in phase 1–3.
- **The GPU/CPU cross-check tests, round-trip tests, style discovery tests.**
  These become the safety net for the refactor.

## What becomes a plugin (and what doesn't)

Not everything should be a plugin — forcing it would be the same mistake in a
new costume. Decision rule: **a thing is a plugin if a fork contributor could
plausibly want to own, disable, or rewrite it alone.**

Good plugin candidates (independent, self-contained):
- Each **filter** family (blur, distort, stylize…) — already catalog-shaped.
- Each **adjustment** — same.
- Each **tool** that owns private state: Brush/Pencil/Eraser/Clone (one paint
  plugin), Shape, Pen, Text, Gradient, Crop. These are exactly the tools the
  assessment lists as bolting state onto the god object.
- **Export/import formats** beyond the core set, later.
- A future **Generate panel** (Stable Diffusion) drops in as one command +
  one tool with zero enum edits — the assessment's Option B becomes cheap.

Stays in the shell (not a plugin — too central or too small):
- Document lifecycle (new/open/save/close), undo/redo, view/zoom, layer tree
  operations, selection boolean state. These are the *substrate* plugins act
  on, not features layered on top.

## The `EditorCtx` seam (the linchpin)

A single mediating handle passed to every plugin entry point. It is the whole
API surface a plugin sees, which is what lets a plugin be built and reviewed
without reading the rest of the app.

```rust
// cshop-ui/src/plugin/ctx.rs  (sketch — exact shape settled in phase 0)
pub struct EditorCtx<'a> {
    doc: &'a mut Document,          // the active document only
    selection: &'a Selection,
    colors: &'a mut ColorState,     // fg/bg swatches
    // queued outputs, applied by the shell after the plugin returns:
    actions: &'a mut Vec<Action>,   // re-use the existing seam
    history_label: Option<&'a str>, // one undo step per gesture
}

impl EditorCtx<'_> {
    pub fn document(&mut self) -> &mut Document { self.doc }
    pub fn selection(&self) -> &Selection { self.selection }
    pub fn emit(&mut self, action: Action) { self.actions.push(action); }
    pub fn begin_undo(&mut self, label: &str) { /* … */ }
}
```

Key property: a plugin **cannot** reach into sibling tools' state or the
window, so it cannot recreate the god-object entanglement. It either mutates
the document through `document()` (recorded as one undo step) or queues an
existing `Action`.

## The trait shapes

```rust
// Command: replaces the menu/shortcut/context triplication.
pub trait Command {
    fn descriptor(&self) -> CommandDescriptor;   // id, label, menu path, shortcut
    fn enabled(&self, ctx: &EditorCtx) -> bool;
    fn run(&self, ctx: &mut EditorCtx);
}

// Tool: replaces the Tool enum + canvas dispatch + options-bar match.
pub trait Tool {
    fn descriptor(&self) -> ToolDescriptor;      // id, name, icon, group, shortcut
    fn on_pointer(&mut self, ev: PointerEvent, ctx: &mut EditorCtx);
    fn options_ui(&mut self, ui: &mut egui::Ui, ctx: &mut EditorCtx) {}
    fn cursor(&self) -> egui::CursorIcon { egui::CursorIcon::Default }
}

// Operation: generalises the filter catalog to adjustments too.
pub trait Operation {
    fn descriptor(&self) -> OpDescriptor;        // id, label, category
    fn params_ui(&mut self, ui: &mut egui::Ui) -> bool;  // returns "changed"
    fn apply(&self, src: &PixelBuffer, ctx: &FilterContext) -> PixelBuffer;
    fn preview_bounded(&self) -> bool { true }
}
```

Tools now **own their state** (`pen draft`, `clone anchor`, `gradient drag`,
`text edit` move off `CShopApp` and into the tool struct). That single change
is what shrinks the god object.

## Registration: one line, discoverable, testable

```rust
// cshop-ui/src/plugin/registry.rs
pub fn register_builtins(reg: &mut Registries) {
    crate::plugins::paint::register(reg);
    crate::plugins::shape::register(reg);
    crate::plugins::pen::register(reg);
    crate::plugins::text::register(reg);
    crate::plugins::filters::register(reg);   // wraps existing Filter catalog
    crate::plugins::adjust::register(reg);     // wraps existing Adjustment catalog
    // add a plugin ⇒ add one line here; nothing else changes
}
```

Mirror the *style discovery test* idea: a test iterates every registered
command/tool/op and asserts invariants (unique ids, unique shortcuts, every
tool has an icon). Adding a plugin is then covered automatically, the same way
styles already are.

## Migration phases (each independently shippable)

Ordered so the tree compiles and tests pass after every phase. No big-bang
rewrite.

### Phase 0 — scaffolding, no behaviour change
- Add `cshop-ui/src/plugin/` (`ctx.rs`, `registry.rs`, traits).
- Introduce `EditorCtx` as a *view* over the existing `CShopApp` fields — no
  fields move yet. Prove one trivial command (`Deselect`) round-trips through
  the registry and still works.
- **Exit test:** full suite green; `Deselect` fired via registry.

### Phase 1 — CommandRegistry (the assessment's #2 recommendation)
- Port the ~55 menu/shortcut/context items to `Command` descriptors. Menu bar,
  `shortcuts.rs` and `context_menus.rs` become *renderers* of one registry
  instead of three parallel tables.
- `Action` and `run()` stay; commands emit `Action`s. This is pure
  de-duplication — lowest risk, highest daily payoff.
- **Exit test:** every old menu item still present (snapshot test of the menu
  tree); shortcuts unchanged.

### Phase 2 — EffectRegistry
- Wrap the existing `Filter` and `Adjustment` catalogs behind `Operation`.
  Filters already work this way; this mainly brings adjustments under the same
  roof so both get menu entries + preview handling for free.
- **Exit test:** GPU/CPU cross-check tests unchanged and green; every filter
  and adjustment still reachable and previews.

### Phase 3 — ToolRegistry (the big one, do last)
- Move each tool's state off `CShopApp` into a `Tool` impl, one tool at a
  time. Start with the simplest (Eyedropper, Hand, Zoom), end with Paint and
  Text.
- Replace the `canvas.rs` pointer `match` with `active_tool.on_pointer(...)`.
- Replace the options-bar `match` with `active_tool.options_ui(...)`.
- Delete the `Tool` enum once the last tool is ported; the toolbar reads
  `ToolRegistry` instead.
- **Exit test:** `input_harness.rs` synthetic-event tests drive each tool
  through the registry and match prior behaviour. This harness is exactly why
  this phase is safe.

### Phase 4 — prune to "basic" (optional, matches the stated goal)
Since the goal is a *basic* editor, a fork may now **disable** heavy plugins by
removing one registration line rather than ripping out code: e.g. drop the 30
filters to a handful, drop Pen/Shape if unwanted. The point of the whole
exercise: scope is now a config of registrations, not a surgery.

## Why this fits a fork better than the monolith

- **One plugin = one folder = one PR.** Debugging a filter no longer means
  reading `chrome.rs`; owning the Shape tool no longer means touching six
  files. This is precisely the "split work per plugin" the fork wants.
- **Compile-time safety is kept.** Registries hold trait objects, but the set
  is assembled in one `register_builtins`, so the compiler still sees every
  built-in. No dynamic-loading ABI, no `unsafe`, honours the pure-Rust rule.
- **The god object dies by attrition,** not by a risky rewrite — its fields
  leave as tools claim them.
- **The existing test philosophy carries over unchanged** and even improves:
  registry-iteration tests cover plugins added later automatically.

## Risks and honest limits

- **Phase 3 is real work.** The god object holds ~60 fields; untangling tool
  state is the bulk of the effort. Do it incrementally, guarded by the input
  harness, and it stays safe. If the fork only wants "modular menus + modular
  filters", phases 1–2 deliver most of the value at a fraction of the cost.
- **This is not runtime plugins.** Third parties still fork-and-compile.
  That is the correct trade for this codebase and this "basic editor" goal;
  the assessment reached the same conclusion (recommendation #7).
- **`EditorCtx` design is load-bearing.** If it leaks too much (`&mut
  CShopApp`), the boundary is fake. Keep it minimal; grow it only when a real
  plugin needs a capability, so it stays an interface rather than a passthrough.
- **`panic = "abort"`** means a panicking plugin still kills the process. For
  built-in Rust plugins that is acceptable (a bug is a bug); revisit only if
  untrusted plugins ever become a goal.

## Recommended first move

Phase 0 + Phase 1 together, on a branch. They touch only `cshop-ui`, change no
behaviour, need no GPU to test, and immediately make "add a menu entry" a
one-line change — proving the registry pattern before committing to the tool
refactor. If it feels right, continue to phases 2–3; if the fork only ever
wants a basic editor, phases 1–2 may be the whole project.
```