# The C-Shop fork: consolidated plan and motivation

This document consolidates the ideas in `docs/fork/`, ordered by how the
thinking actually developed (by file creation date on 2026-08-30). Each note
built on the last, so reading them as a sequence shows not just *what* the plan
is but *why* it arrived there. The source notes, in order:

1. [`fork-assessment.md`](fork-assessment.md) — 02:37 — is the code sound and extensible?
2. [`modular-rewrite.md`](modular-rewrite.md) — 02:45 — how to make it modular
3. [`build-vs-rebuild.md`](build-vs-rebuild.md) — 02:54 — build on it or start fresh?
4. [`findings-report.md`](findings-report.md) — 02:59 — the consolidated technical report
5. [`rapidraw-integration.md`](rapidraw-integration.md) — 03:06 — delegating the raw workflow
6. [`beginner-first-direction.md`](beginner-first-direction.md) — 03:27 — the intuition-first product direction

All claims about the code were verified against the tree at commit `a83e6cb`:
`cargo check --workspace --tests` passes, `cshop-core`/`cshop-io` tests pass,
`cshop-ui` tests need a real GPU (segfault headless), and there is no CI config.

---

## The motivation, in one paragraph

The goal is a **basic, modular, AI-assisted image editor for a non-technical,
AI-first audience** doing three things: txt2img creation, img2img/photo
enhancement via Stable Diffusion, and basic photo edits (masks + adjustments).
C-Shop is a candidate foundation because its expensive, hard-to-build parts (a
tested compositor, layered file formats, and a sandboxed AI-editing server) are
exactly what the product needs, while its weak parts (internal modularity and
UI scope) are cheap to fix. The investigation below works out whether that
premise holds, how to fix the weak parts, and — in its later passes — that the
*real* product may be a separate intuition-first frontend with C-Shop playing a
supporting, downstream role.

---

## 1. Is C-Shop sound and extensible? (the assessment)

The first pass asked whether an AI-written codebase could carry a modular system
beyond a plain image editor. The answer: **the code is sound; the structure is a
compiler-guarded monolith.**

- **Quality is above average.** Documented decisions (`docs/ARCHITECTURE.md`
  records *why*, including bugs), CPU-vs-shader pixel cross-checks, synthetic
  input-harness UI tests, fuzz-tested tagged-chunk file formats, and bounded
  preview cost. The concern was never quality — it was structure.
- **Clean layering, one god object.** Five crates with a genuine one-way
  dependency graph (`cshop-core` pure and tested, `cshop-gpu`, `cshop-io`,
  `cshop-ui`, `cshop-app`). The design centre is a real **command pattern**: an
  `Action` enum (~120 variants) that every surface emits, funnelling all
  mutation through one `run()` match. But `CShopApp` (~3,800 lines, ~60 fields)
  is a flat god object that owns every tool's private state, and `run()` is a
  single ~660-line match.
- **Closed enums everywhere.** Filters are catalog-driven (one variant → a menu
  entry for free — the *good* pattern), but everything else is hand-written
  across three surfaces (menu bar, shortcuts, context menus). Tools are a closed
  19-variant enum touching ~8 compile sites to extend. There are no registries,
  trait objects, or dynamic loading. Extension is **compile-time, not
  run-time.**
- **The AI story is already built externally.** The MCP/script server is a
  sandboxed, session-holding, headless frontend with `place` (PNG → layer),
  `export` (region → file), selections-as-masks, and image-carrying results.
  That *is* the img2img loop, spoken over a socket and tested — an external
  orchestrator can drive SDNext/ComfyUI today with zero editor changes.

Conclusion: not a fragile vibe-code. A disciplined monolith whose bias toward
closed sets makes in-process work tedious but safe, and which leaves the outside
doors (script, MCP, styles, formats) wide open. The first move should be a
bridge over MCP, not a refactor.

## 2. How to make it modular (the rewrite plan)

The second pass defined what "modular" should mean and how to get there **without
a big-bang rewrite**. It explicitly rejects dynamically-loaded `.so`/`.dll`
plugins (they fight the pure-Rust constraint, `panic = "abort"`, and the lack of
an ABI). Instead:

> **In-process plugins: self-contained Rust modules that register into small
> runtime registries, so a feature is one folder you can build, test, and debug
> in isolation — never an edit spread across eight files.**

The design replaces the three closed extension points with three registries plus
one mediating context:

| Registry | Replaces | A plugin provides |
|---|---|---|
| `CommandRegistry` | menu/shortcut/context triplication | one descriptor: id, label, menu path, shortcut, `enabled`, `run` |
| `ToolRegistry` | the `Tool` enum + 8 touch points | a `Tool` trait impl owning its own state + pointer handlers |
| `EffectRegistry` | generalise the good filter pattern | an `Operation`: params + `apply` |

The linchpin is **`EditorCtx`** — the single mediated handle a plugin sees
(active document, selection, colours, an `emit(Action)` queue, an undo label). A
plugin cannot reach sibling state or the window, so it *cannot* recreate the god
object; tools move their state onto themselves and the god object dissolves by
attrition. Crucially, the `Action`/`run()` seam **stays** — commands emit
`Action`s, so the enum stops *growing* without being torn out. `cshop-core`,
`cshop-gpu`, `cshop-io`, styles, script, and MCP are untouched in the early
phases.

**Migration phases (each independently shippable, tree green after each):**
- **Phase 0** — scaffolding: add the `plugin/` module and `EditorCtx` as a view
  over existing fields; route one trivial command through the registry.
- **Phase 1** — `CommandRegistry`: port ~55 menu/shortcut/context items so the
  three surfaces become renderers of one registry. Lowest risk, highest daily
  payoff; makes "add a menu entry" one line.
- **Phase 2** — `EffectRegistry`: wrap the existing `Filter`/`Adjustment`
  catalogs behind `Operation`.
- **Phase 3** — `ToolRegistry`: move each tool's state off `CShopApp`, simplest
  first, guarded by the input harness.
- **Phase 4** — trim to "basic": scope becomes which plugins you register.

If the fork only wants "modular menus + modular filters", phases 1–2 deliver
most of the value.

## 3. Build on it, or start fresh? (the product decision)

The third pass reframed the question as a product one and reached a firm verdict:
**build on C-Shop; do not start fresh.**

- **txt2img + img2img** are the codebase's strongest fit — the plumbing exists
  and is tested over a socket. A rewrite would spend its first month rebuilding
  it.
- **Masks + adjustments** are present and cross-checked. A straight win.
- **The muscle-memory argument.** For an incoming user, shortcuts and menu
  geography *are* the product — this is why editors lose users to "it's not
  Photoshop". C-Shop is already Photoshop-shaped by default (PS shortcut scheme,
  Image/Layer/Select/Filter menus, single-window). GIMP is the cautionary tale;
  Krita the encouraging one (it won painters by nailing *their* muscle memory).
  Since the target audience is AI-first creators — neither PS pros nor painters —
  the PS-shaped defaults are "familiar enough" without being a parity promise.
- **A rewrite discards ~38k lines** of tested Rust (compositor, 27 blend modes,
  `.cshop` + PSD codecs, the MCP/script harness) to avoid a UI that is already
  the shape you want and whose real problems are fixable in place via the bounded
  phased plan. That trade is strongly negative.

The plan is to **trim, not rebuild** (un-register heavy plugins rather than
delete code) and **add** four small pieces: a `cshop-net` crate (extract the MCP
HTTP/JSON, add a client), a GUI task runner (no `thread::spawn` exists in
`cshop-ui`, so a 20s generation would freeze the window), a config store, and
layer provenance. A Generate/Enhance panel then lands as one command + one tool
after Phase 1.

## 4. The one real limitation, and how to retire it (raw)

Across passes 3 and 4, the single genuine limitation surfaced and got resolved:

- **The core is 8-bit sRGB** (`PixelBuffer` is `Vec<Rgba8>`) with **no raw
  decoder**. Opening and *developing* a .CR2/.NEF/.ARW is not supported and not
  cheap — true raw implies a 16-bit/float pipeline touching the pixel core, the
  compositor, and every filter.
- **Do not attempt a 16-bit core rewrite.** Instead, **delegate raw development
  to RapidRAW** (CyberTimon/RapidRAW) — a mature Rust/Tauri Lightroom-style raw
  developer with a 32-bit pipeline, full raw decode (rawler/dnglab), AI masks
  (SAM 2 / U-2-Net / Depth Anything v2), and ComfyUI generative edits. It has
  **no layers, type, shapes, or compositing** — exactly what C-Shop provides, so
  the two are complementary, not competitors.
- **The seam is the developed 8-bit image.** RapidRAW develops the raw; C-Shop
  composites and does agentic/generative work on the result. Nothing is lost
  because SD img2img is itself an 8-bit round trip. Two front doors, one handoff:
  *"making something"* → C-Shop; *"fixing a photo"* → RapidRAW.
- **Two hard constraints.** (1) **Licensing:** C-Shop is MIT/Apache-2.0,
  RapidRAW is AGPL-3.0 — never copy its code in; keep integration arm's-length
  (separate processes / file handoff). (2) **Don't reinvent the ComfyUI glue** —
  when C-Shop builds its own Generate panel, mirror RapidRAW's middleware
  *architecture* (a local sidecar over HTTP), not its code.
- **Integration, cheapest first:** (A) "Open With" file handoff — zero code,
  ship first; (B) shared workspace folder feeding C-Shop's `place` loop; (C)
  shared ComfyUI later, using RapidRAW's connector only as a design reference.

This delegation retires the only weak row in the fit scorecard and makes the
build-on-C-Shop decision stronger by shrinking scope.

## 5. Two near-term feature bets

The consolidated technical report also sized two features worth scheduling early:

- **Liquify** — the one push-pixels tool the audience will expect, and it fits
  unusually well. `cshop-core/src/filters/distort.rs` already implements
  distortions as a rayon-parallel **backward map** over an f32 `Plane` with
  bilinear sampling, and filter dialogs already preview on a downscaled proxy and
  commit at full resolution. So Liquify is a *displacement field* driving the
  existing `warp()`, accumulated from brush strokes. After the `ToolRegistry` it
  lands as a self-contained plugin (state on the tool, a `liquify` core module,
  proxy preview, one undoable commit). Small-to-medium, low-risk, CPU-testable.
- **Cloud/mobile** — the Lightroom-style "sync + mobile app editing the same
  catalog" expectation is real but is an **infrastructure track to design
  toward, not build now.** The project already leans the right way: `cshop-app`
  runs headless with sandboxed sessions, edits are expressible as
  `Action`s/scripts (data, not pixels), a software-Vulkan render path exists, and
  `.cshop` is forward-compatible. Guidance for now: keep edits data-shaped, keep
  the format compatible, design `cshop-net`/the task runner so the same transport
  could reach a remote render service, and do not stand up accounts/storage/mobile
  until the desktop fork is mature.

## 6. The pivot: an intuition-first product, C-Shop downstream

The final pass is a **separate product direction** and the most consequential
shift in the whole sequence. The earlier notes treat C-Shop as a
Photoshop-familiar editor made modular. This one asks what tool best serves a
specific person — an expert makeup artist whose ADHD/dyslexia have sharpened a
powerful *visual intuition* — and reaches a different conclusion.

- **The right tool is not a trimmed C-Shop.** Reducing menus yields a *crippled
  Photoshop*, whose interaction grammar (pick a tool → act → tune options) is
  the very thing that excludes her. The tax is not menu count; it is the
  **translation from intuition to procedure**. A goal-first app inverts the
  grammar: *pick an outcome → optionally point at where → see the result.* That
  is a different UI.
- **That UI already exists as a working prototype** — the **Visual Look Editor /
  Recipe Factory** in the user's `~/Projects/ai-image` project. It is a
  client-side web app whose design (verb cards, "Visual Locks" to keep
  pose/face/light while changing one thing, swatch recognition over text recall,
  non-destructive branching + implicit "Memories", a human-readable Recipe Card)
  independently matches every accessibility principle. What it lacks is a
  generation backend — that is its Phase 2.
- **Generation belongs to ComfyUI, not C-Shop.** The Visual Locks map onto
  ComfyUI conditioners (keep-pose → ControlNet/OpenPose, keep-face →
  PuLID/IP-Adapter, keep-light → IC-Light) — which C-Shop cannot host. So the
  Look Editor drives ComfyUI directly, and **C-Shop is a clean *downstream*
  editor** the result can be exported into ("edit further"), not the engine.
- **Design for an expert, not a beginner.** Her makeup vocabulary *is* the
  control surface and the product's moat — co-author the skill library with her,
  in her words. Automate the technical (masks, seeds, denoise, versions), expose
  only the tasteful (which region, which direction, which result). Fit the fast,
  divergent, non-linear rhythm; never a linear wizard.
- **The novel part is reference-and-recreate.** Ingesting a sample image and
  extracting *how to recreate it in the real world* (light placement, camera
  config, studio vs. outdoor) into a physical **recreation Recipe Card** is
  something no existing tool does. It is a VLM + templating problem living
  entirely in `ai-image`, needing nothing from C-Shop.
- **Workflow 1 wins first.** Treat generation as a **pre-production reference**:
  she designs the shoot and gets a reference image + recreation recipe; the real
  shoot happens; the human editing phase (raw develop → retouch) is separate and
  conventional. The generated image is a *brief, not a deliverable*, so 8-bit
  output, identity drift, and bad hands do not matter. This is low-effort (mostly
  already built) and avoids the all-in-one Workflow 2, which re-introduces every
  raw/compositing problem the other docs deliberately delegated away.

**C-Shop's role in this direction** is narrow but real: it is where *you* retouch
the developed photograph, **and where she does the final crop-and-export**. That
delivery step is a concrete, bounded, her-facing win — C-Shop already has the
primitives (JPEG-at-quality, Lanczos-3 resize, crop-to-aspect, scripting); what's
missing is three aspect presets (4:5, 1.91:1, 9:16) and a one-click "export
delivery set" (agency master + social crops). It doubles as the fork's first
`CommandRegistry` feature.

## The four surfaces

The final architecture is four surfaces, layered by how much control the user
wants, meeting only at file/HTTP boundaries:

| Surface | Audience | Engine behind it |
|---|---|---|
| **Look Editor** (`ai-image`) | the makeup artist; intuitive creatives | ComfyUI (+ VLM for recipes) |
| **C-Shop fork** | you; prosumers; Photoshop refugees | its own compositor + MCP AI loop |
| **RapidRAW** | photographers | its own 32-bit raw pipeline |
| **ComfyUI** | (never user-facing here) | itself |

---

## Consolidated order of work

Merging the recommendations across all six notes, earliest-value first:

1. **Bridge SD over MCP** (Option A) — zero editor changes; proves the loop from
   outside.
2. **Wire the RapidRAW handoff** — "Open With" + shared workspace; delivers the
   raw/photographer flow with near-zero code.
3. **Phase 0 + Phase 1** (command registry) on a branch — UI-only, no behaviour
   change, no GPU needed; makes "add a menu entry" one line.
4. **Build the delivery export panel** as the first command-registry feature —
   add 4:5 / 1.91:1 / 9:16 presets + one-click "export delivery set".
5. **`cshop-net` crate** — extract HTTP/JSON, add a client; design the transport
   so it could also reach a remote render service later.
6. **GUI task runner** with document identity and loop wakeup — the gap
   everything else falls into; build it *before* any Generate panel.
7. **Config store + layer provenance.**
8. **Generate/Enhance panel** as a plugin (mirror RapidRAW's ComfyUI-middleware
   architecture, not its AGPL code).
9. **Phase 2 (effects) and Phase 3 (tools)** as appetite allows; **Phase 4** trims
   scope.
10. **Liquify plugin** — the first non-trivial tool after the registry; reuses the
    existing warp/sample/proxy/history seams.
11. **Cloud/mobile track** — design toward it, build only once the desktop fork is
    mature; keep edits data-shaped and the format compatible.
12. **Do not pursue dynamic plugins.** Compile-time exhaustiveness is a feature
    here; a crate registered into the registries is the idiomatic extension path.

**On the intuition-first track (separate, personal-value first):** wire the Look
Editor to ComfyUI (its Phase 2) first; build the recreation Recipe Card second;
build the C-Shop delivery export panel third; refine the skill vocabulary with
her throughout.

## Bottom line

Build on C-Shop rather than starting fresh: its strongest parts are the expensive
ones the product needs, and its weakest parts (modularity, scope) are fixable in
place through a bounded, phased in-process-plugin refactor that keeps the
compile-time safety and the command-pattern seam intact. Delegate raw to
RapidRAW and generation to ComfyUI rather than absorbing either. And recognise
that the highest-value product for the intended user is the already-built
intuition-first Look Editor, with C-Shop as a supporting downstream editor and
the home of her delivery/export step — not the engine at the centre.
