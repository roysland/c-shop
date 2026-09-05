# Fork assessment: modularity and extensibility

A review of C-Shop from the point of view of someone who likes the editor and
the UI, but needs to know whether the structure can carry it beyond a plain
image editor. Written 2026-08-30 against commit `a83e6cb`. Verified on this
fork: `cargo check --workspace --tests` passes clean, `cshop-core` and
`cshop-io` tests pass, `cshop-ui` tests segfault without a real GPU adapter.

## The questions

1. **Is the code sound enough to build on?** The premise was that AI-written
   code is probably fine, but the plan and structure might not be.
2. **How is the project organised, and is the layering real?**
3. **How easy is it to add a menu entry?**
4. **How easy is it to add a tool?**
5. **Is there a plugin story — anything that lets an outsider add features
   without editing the core?**
6. **Could this carry Stable Diffusion workflows** — calling an SDNext or
   ComfyUI API, so the app becomes a surface for generation rather than just
   an image editor?

## The short answers

1. Yes. The code quality is above average for any codebase: documented
   decisions, cross-checked GPU/CPU tests, real performance discipline. The
   concern was structure, and the structure is mostly good but has one strong
   bias: **everything is a closed, statically-known enum, and there is no
   dynamic extension surface anywhere.**
2. Five crates with a genuinely clean dependency layering and a pure,
   well-tested core. The `cshop-ui` crate contains a 3,800-line god-object
   (`CShopApp`) that owns all tool state.
3. Two patterns coexist. Generic commands are hand-written per item across
   three surfaces (menu bar, shortcuts, context menus). Filters, by contrast,
   are **catalog-driven** — one enum variant yields a menu entry for free. The
   filter pattern is the right one, but it is applied only to filters.
4. About eight compile-guarded touch points: the `Tool` enum, metadata
   matches, the toolbar group table, the pointer-dispatch match in
   `canvas.rs`, the options bar, and state fields bolted onto `CShopApp`.
   Safe (the compiler enumerates every site) but not extensible — there is no
   `Tool` trait anything could implement.
5. No. No registries, no trait objects, no dynamic loading. The closest real
   plugin mechanism is the on-disk `.style` system, which composes existing
   commands only.
6. **The hard part is already built, via the external path.** The repo ships
   an MCP server over hand-written HTTP with sessions, a workspace sandbox,
   `place` to ingest a PNG as a layer, selections-as-masks, and `export`. An
   SDNext or ComfyUI workflow can orchestrate against that today with zero
   changes to the editor. An *in-app* Generate panel needs four things that
   do not exist yet: an async task layer, an HTTP client, a config store, and
   layer provenance. None is large; see [Stable Diffusion
   integration](#stable-diffusion-integration).

---

## Detailed findings

### 1. Code quality: the premise holds

The parts most likely to rot in a vibe-coded project are handled with
unusual care:

- `docs/ARCHITECTURE.md` records *why* decisions were made, including the
  bugs they caused (the double-linearisation gamma bug, per-pixel LUT rebuilds
  costing 11.4 s).
- Blend modes and adjustments are implemented twice (CPU and shader) and
  compared pixel by pixel.
- `input_harness.rs` drives the real `update` loop with synthetic events
  because widget hit-testing bugs had passed every prior test.
- The `.cshop` format is hand-serialized with tagged chunks so readers skip
  what they don't recognise — files outlive the code.
- Preview cost is *bounded* rather than proportional to document size, with
  tests comparing preview output to applied output.

Repo state on this machine: compiles clean including tests; core/io tests
pass. The UI test suite requires a working GPU adapter (SIGSEGV headless
here), and there is no CI configuration in the repo despite the docs
referring to CI. Both are worth fixing before building on top.

### 2. Structure: clean layering, one god object

```
cshop-core   document model, pixels, history, filters, adjustments   (no GPU/UI/IO deps)
cshop-gpu    wgpu compositor
cshop-io     PSD + .cshop codecs
cshop-ui     egui interface (~30 files)
cshop-app    window, script harness, MCP server
```

The layering is real, not decorative: `cshop-core` stays pure and fast to
test, and the crate boundaries are respected. The design centre is the
**command pattern**: `commands.rs` defines an `Action` enum (~120 variants);
menus, shortcuts, toolbar buttons and panel controls all *emit* actions, and
all mutation funnels through one `run(&mut self, action)` match in
`app.rs:737`. That is a genuinely good central seam.

Two structural flaws sit around it:

- `CShopApp` (`cshop-ui/src/app.rs`, 3,813 lines) is a flat god object of
  ~60 state fields. Each tool's state — pen draft, clone anchor, gradient
  drag, text edit — lives on the app, not on the tool.
- `run()` is a single ~660-line match. Fine for one maintainer, a merge
  magnet for anyone else.

### 3. Menu entries: one good pattern, one bad one

**Filters (good).** The Filter menu is populated generically from the
catalog (`chrome.rs:683` — `Filter::all_defaults()` grouped by
`Category::ALL`). Adding a filter variant yields its menu entry, label and
"…" suffix for free.

**Everything else (bad).** `menu_bar` in `chrome.rs` is ~330 lines of
hand-written egui closures with ad-hoc enabled predicates and shortcut hints
pulled from a parallel table in `shortcuts.rs`, whose 55 actions are then
referenced *again* in `context_menus.rs`. One menu item, three surfaces, no
single source of truth.

The fix is to generalise the filter pattern into a command registry: one
descriptor per command — id, label, shortcut, `enabled`, `run` — driving all
three surfaces.

### 4. Tools: a closed enum, compiler-guarded

`Tool` is a 19-variant enum in `tools.rs`. Adding one touches:

| Site | What |
|---|---|
| `tools.rs` | enum + `name`/`glyph`/`is_implemented`/`is_selection_tool`/`uses_brush` |
| `tools.rs` | `TOOL_GROUPS` entry (toolbar slot + shortcut) |
| `canvas.rs:598` | pointer dispatch match (down/drag/up) |
| `canvas.rs:1137` | cursor mapping |
| `chrome.rs` | options bar controls |
| `app.rs` | state fields for the tool |
| tests | `tools.rs` invariant tests |

Rust's exhaustive matching means the compiler lists every one of these —
which makes the work *safe and tedious* rather than *fragile*. But there is
no `Tool` trait, no lifecycle object; nothing an outsider could implement
without editing the enum. That is the central fact of this codebase: the
extension points are compile-time, not run-time.

### 5. Plugins: none, and honestly assessed

No registries, no trait objects at any boundary, no dynamic loading. The
`.style` system is the only open, on-disk extension mechanism — styles live
in `~/.config/cshop/styles`, are parameterised, validated, and composable.
It is a lovely macro system, but it can only recombine existing commands.

The script harness (`--script`, `--run`) is the second frontend and the
project's biggest strategic asset: the same editor runs headless with
sessions, a workspace sandbox, machine-readable reports, and `measure` for
callers that cannot see the canvas. Over MCP (`--serve`, hand-written
HTTP/JSON-RPC, no SDK, no async runtime), a tool result can carry a picture,
closing the describe→draw→look→correct loop over a network.

Two caveats:

- The docs' claim that "anything the editor gains is reachable [in script]
  the same day" is aspirational. `script.rs` (2,037 lines) is a *parallel*
  implementation against core, not a bridge to `Action` (filter names are
  re-parsed at `script.rs:1499`). GUI and harness can drift by structure.
- The MCP server replaces the window; the GUI itself serves nothing.

### 6. Stable Diffusion integration

#### Option A — external orchestration over MCP (available today)

The SD workflow is an orchestration problem, and the pieces already form a
complete pipeline:

1. Orchestrator (agent, shell script, or ~100-line bridge binary) calls
   SDNext (`/sdapi/v1/txt2img`, `/img2img`) or ComfyUI (`/prompt` +
   `/history` + output images) over plain HTTP.
2. Output PNG is written into the served workspace.
3. `cshop --serve` session runs `place out.png` — it lands as a real layer
   among the user's layers.
4. For img2img/inpaint round-trips: `export` the layer or selection, feed
   the file to SD, `place` the result. Selections-as-masks and channels
   already exist, so the mask path is built.

Effort: near-zero inside the editor. Modularity risk: none — no closed enum
is touched. This is the recommended first integration and the right shape
for a *fork* of this project: treat the editor as the MCP server, and grow
generation as clients and scripts around it.

#### Option B — in-app Generate panel (needs four missing layers)

A tool or panel that calls the SD API itself, with progress, requires:

1. **Async task layer in the GUI.** There is no `thread::spawn`/mpsc
   anywhere in `cshop-ui`; heavy filters run synchronously under rayon. A
   20-second generation would freeze the editor, and the window event loop
   deliberately idles, so it needs a wake-on-event hook. Tasks must key to
   document *identity* (docs live in a `Vec<DocView>` with an active index —
   the active document can change or close while a request is in flight) and
   land as one undoable step, which the per-gesture history already
   supports.
2. **HTTP client.** The MCP HTTP code (`cshop-app/src/mcp/`) is server-only;
   the JSON codec there is private to that crate. Extract both into a small
   `cshop-net` crate and add a client. For `localhost:7860`/`8188` plain
   `std::net` suffices and honours the pure-Rust constraint; remote/TLS
   means rustls — still pure Rust, but the first heavyweight dependency.
3. **Config store.** No settings file exists anywhere (styles search
   `~/.config/cshop`, but nothing reads a config). Server URLs, models,
   sampler defaults, prompt presets need a home.
4. **Layer provenance.** `Layer` has no metadata bag; "which layer came from
   which prompt/job/model" has nowhere to live. Small model change; the
   tagged-chunk project format absorbs it well.

The generation payload itself has good seams already: a layer or merged
region exports through existing codecs, and `place`ing a result is an
existing command.

### 7. Additional risks noted in passing

- `panic = "abort"` in the release profile (`Cargo.toml:52`) — a panic in a
  served session or a future network response handler kills the whole
  process; reconsider if serving becomes multi-user.
- No CI config, and UI tests are GPU-dependent (they segfault headless).
- Hard constraint ("no system packages; X11 via dlopen, Vulkan via ash,
  everything pure Rust") shaped every dependency decision. Any extension
  inherits it — relevant when evaluating HTTP/TLS/async options.

## Recommended order of work

1. **Bridge SD over MCP first** (Option A). Zero editor changes; proves the
   workflow and exercises the sandbox/sessions from the outside.
2. **Command registry.** Generalise the filter catalog pattern so menu bar,
   shortcuts and context menus share one source. Makes "add menu entry" a
   one-line change and is the natural attach point for new commands like
   `Generate…`.
3. **`cshop-net` crate.** Extract `json.rs`/`http.rs`/`base64.rs` from
   `cshop-app/src/mcp/`, add a client. Prerequisite for all in-app network
   features.
4. **GUI task runner** with document identity and loop wakeup. The gap
   everything else falls into; do it before writing any SD UI code.
5. **Config store + layer metadata bag.** Small, needed for the panel.
6. **`Tool` trait refactor.** The largest change; only worth it if several
   exotic tools are planned. A Generate panel can live as a dialog + actions
   and skip it entirely.
7. **Do not pursue dynamic plugins.** Closed enums with compile-time
   exhaustiveness are a *feature* here — every call site is compiler-
   enumerated, and a `cshop-sd` crate registered into the existing enums is
   the idiomatic Rust extension path for this design. Just keep the docs
   honest: "extensible" means fork-and-compile, or drive it over
   MCP/script.

## Conclusion

This is not a fragile vibe-code. It is a disciplined, compiler-guarded
monolith with unusual architectural maturity — a real command pattern, a
pure tested core, and two headless frontends nobody asks a Photoshop clone
to have. Its design bias is closed sets everywhere; that bias makes
in-process plugin work tedious but safe, and it happens to leave the outside
doors (script, MCP, styles, formats) wide open. For the stated goal, the
fork's first move is not a refactor at all: it is a bridge script against
`--serve`, after which the registry, net crate and task layer above are each
small, independent, and clearly worth having.
