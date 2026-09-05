# C-Shop fork: consolidated findings report

A single report gathering every finding from the three-pass investigation of
C-Shop as the basis for a **modular, AI-assisted image editor** aimed at
**non-technical users** doing txt2img creation, img2img enhancement, and basic
photo edits. It consolidates and supersedes the working notes in this folder:
[`fork-assessment.md`](fork-assessment.md) (modularity/extensibility),
[`modular-rewrite.md`](modular-rewrite.md) (the plugin design),
[`build-vs-rebuild.md`](build-vs-rebuild.md) (product decision), and
[`rapidraw-integration.md`](rapidraw-integration.md) (raw delegation).

Written 2026-08-30 against the same tree. Claims about the code were verified
in source; the decisive ones are cited inline.

---

## Headings

1. [Executive summary](#1-executive-summary)
2. [Scope and audience](#2-scope-and-audience)
3. [Is the code sound enough to build on?](#3-is-the-code-sound-enough-to-build-on)
4. [How the project is structured](#4-how-the-project-is-structured)
5. [The two structural flaws](#5-the-two-structural-flaws)
6. [Extensibility today: closed enums everywhere](#6-extensibility-today-closed-enums-everywhere)
7. [The modular rewrite plan](#7-the-modular-rewrite-plan)
8. [Migration phases](#8-migration-phases)
9. [Comparison with Krita and GIMP](#9-comparison-with-krita-and-gimp)
10. [The muscle-memory problem](#10-the-muscle-memory-problem)
11. [Fit for the AI-assisted workflow](#11-fit-for-the-ai-assisted-workflow)
12. [The raw / photographer caveat](#12-the-raw--photographer-caveat)
13. [RapidRAW: delegating the raw workflow](#13-rapidraw-delegating-the-raw-workflow)
14. [The Liquify tool: a near-term priority](#14-the-liquify-tool-a-near-term-priority)
15. [Cloud and mobile: a future infrastructure track](#15-cloud-and-mobile-a-future-infrastructure-track)
16. [Build vs. rebuild decision](#16-build-vs-rebuild-decision)
17. [What to keep, trim, and add](#17-what-to-keep-trim-and-add)
18. [Risks](#18-risks)
19. [Recommended order of work](#19-recommended-order-of-work)
20. [Appendix: verified facts and citations](#20-appendix-verified-facts-and-citations)

---

## 1. Executive summary

**Build on C-Shop; do not start a new project.** It is an unusual codebase
whose *weakest* area — internal modularity and UI scope — is the cheap-to-fix
part, and whose *strongest* area — a tested Vulkan compositor, layered file
formats, and a sandboxed AI-editing server — is exactly the expensive part the
product needs and a rewrite would have to reproduce.

Three findings drive everything:

- **The code is sound.** Above-average discipline: documented decisions,
  GPU-vs-CPU pixel cross-checks, synthetic-input UI tests, fuzz-tested file
  formats. The concern was never quality; it was structure.
- **The structure is a compiler-guarded monolith.** Clean crate layering and a
  real command pattern, but *closed enums everywhere* and one 3,800-line god
  object. Adding a feature touches ~8 compile sites. This is fixable in place.
- **The AI workflow is mostly already built.** The MCP/script server is the
  img2img loop — sandboxed sessions, place-a-PNG-as-a-layer, export-a-region,
  see-what-you-drew — and it is tested over a real socket.

The one genuine limitation is **raw photography**: the pixel core is 8-bit
sRGB and there is no raw decoder. Rather than a costly 16-bit core rewrite,
**delegate raw development to a dedicated tool (RapidRAW)** and keep C-Shop for
compositing + AI editing — see [§13](#13-rapidraw-delegating-the-raw-workflow).
That handoff retires the limitation and shrinks C-Shop's scope, so the
build-on-C-Shop decision only gets stronger.

## 2. Scope and audience

The goal is **not** a Photoshop clone. It is a *basic* modular image editor for
a **non-technical, AI-first audience** with three workflows:

1. **txt2img** — create an image from a prompt.
2. **img2img / enhance** — improve an image with Stable Diffusion.
3. **Photographer flow** — open a photo, do basic masks and adjustments, then
   enhance with SD. The raw *develop* stage of this flow is delegated to
   RapidRAW ([§13](#13-rapidraw-delegating-the-raw-workflow)); C-Shop owns the
   compositing and AI-editing stage.

C-Shop's scope is therefore **a basic modular compositing + AI-assisted
editor**; raw development is explicitly out of scope and handed off upstream.
"Modular" is a means to an end: let a fork split work, debugging and features
per-plugin instead of across the whole codebase.

## 3. Is the code sound enough to build on?

Yes. The parts most likely to rot in vibe-coded projects are handled with care:

- `docs/ARCHITECTURE.md` records *why* decisions were made, including bugs they
  caused (double-linearisation gamma bug; per-pixel LUT rebuilds costing 11.4s).
- Blend modes and adjustments are implemented **twice** (CPU + shader) and
  compared pixel by pixel (worst divergence 0.84/1.1 of one 8-bit level).
- `input_harness.rs` drives the real update loop with synthetic pointer/keyboard
  events, catching widget hit-testing bugs earlier tests missed.
- The `.cshop` format uses tagged chunks so readers skip what they don't know;
  both layered formats are round-trip- and fuzz-tested.
- Preview cost is *bounded*, not proportional to document size.

Repo state on this machine: compiles clean including tests; core/io tests pass.
UI tests need a real GPU (segfault headless). No CI config in the repo.

## 4. How the project is structured

Five crates, dependencies pointing one way:

```
cshop-core   document model, pixels, history, filters, adjustments  (no GPU/UI/IO)
cshop-gpu    wgpu/Vulkan compositor
cshop-io     PSD + .cshop codecs
cshop-ui     egui interface (~30 files)
cshop-app    window, script harness, MCP server
```

The layering is real, not decorative. The design centre is the **command
pattern**: `commands.rs` defines an `Action` enum (~120 variants); every
surface *emits* actions and all mutation funnels through one
`run(&mut self, action)` match in `app.rs`. That single seam is the codebase's
best feature and the natural attach point for new work.

## 5. The two structural flaws

1. **The god object.** `CShopApp` (`cshop-ui/src/app.rs`, ~3,800 lines) is a
   flat object of ~60 fields. Each tool's private state — pen draft, clone
   anchor, gradient drag, text edit — lives on the app, not on the tool.
2. **The mega-match.** `run()` is a single ~660-line match. Safe for one
   maintainer, a merge magnet for a team.

Both are the *execution failure* of the original "move away from monolithic"
intent: the layering is modular, but the interaction layer is not.

## 6. Extensibility today: closed enums everywhere

- **Menus:** two patterns coexist. Filters are **catalog-driven** (one enum
  variant yields a menu entry for free — the *good* pattern). Everything else
  is hand-written across three surfaces (menu bar, `shortcuts.rs`,
  `context_menus.rs`) — one item, three sources of truth.
- **Tools:** a closed 19-variant `Tool` enum. Adding one touches ~8 sites
  (enum + metadata, `TOOL_GROUPS`, canvas pointer dispatch, cursor map, options
  bar, app state fields, tests). Compiler-enumerated, so *safe but tedious* —
  there is no `Tool` trait an outsider could implement.
- **Plugins:** none. No registries, no trait objects at boundaries, no dynamic
  loading. The `.style` system is the only open extension mechanism, and it can
  only recombine existing commands.

Conclusion: the extension points are **compile-time, not run-time**.

## 7. The modular rewrite plan

"Modular" here should **not** mean dynamically-loaded `.so`/`.dll` plugins —
that fights the pure-Rust constraint, `panic = "abort"`, and the lack of an ABI
story, and is overkill for a basic editor. It should mean:

> **In-process plugins: self-contained Rust modules that register into small
> runtime registries, so a feature is one folder you can build, test and debug
> in isolation — never an edit spread across eight files.**

Replace the three closed extension points with three registries plus one
mediating context:

| Registry | Replaces | A plugin provides |
|---|---|---|
| `CommandRegistry` | menu/shortcut/context triplication | one descriptor: id, label, menu path, shortcut, `enabled`, `run` |
| `ToolRegistry` | the `Tool` enum + 8 touch points | one `Tool` trait impl owning its own state + pointer handlers |
| `EffectRegistry` | *(generalise the good filter pattern)* | one `Operation`: params + `apply` |

The linchpin is **`EditorCtx`** — the single mediated handle a plugin sees
(active document, selection, colours, an `emit(Action)` queue, an undo label).
A plugin cannot reach sibling tools' state or the window, so it *cannot*
recreate the god object. Tools move their private state onto themselves, and
the god object dissolves by attrition rather than by a risky rewrite.

Crucially, the `Action`/`run()` seam **stays** — commands emit `Action`s; the
enum stops *growing* without being torn out. `cshop-core`, `cshop-gpu`,
`cshop-io`, styles, script and MCP are untouched in the early phases.

## 8. Migration phases

Each phase compiles and passes tests on its own; no big-bang rewrite.

- **Phase 0 — scaffolding.** Add `plugin/` module and `EditorCtx` as a view
  over existing fields; route one trivial command through the registry.
- **Phase 1 — CommandRegistry.** Port ~55 menu/shortcut/context items to one
  registry; the three surfaces become renderers of it. Lowest risk, highest
  daily payoff; makes "add a menu entry" a one-line change.
- **Phase 2 — EffectRegistry.** Wrap the existing `Filter`/`Adjustment`
  catalogs behind `Operation` so both get menus + preview handling uniformly.
- **Phase 3 — ToolRegistry.** Move each tool's state off `CShopApp` into a
  `Tool` impl, simplest first; replace the canvas pointer match and options-bar
  match with dispatch through the active tool. Guarded by `input_harness.rs`.
- **Phase 4 — trim to "basic".** Scope becomes which plugins you register;
  drop filters/Pen/Shape by removing registration lines, not by surgery.

If the fork only wants "modular menus + modular filters", phases 1–2 deliver
most of the value at a fraction of the cost.

## 9. Comparison with Krita and GIMP

| | Photoshop | Krita | GIMP | C-Shop (today) |
|---|---|---|---|---|
| Default shortcuts | — | mostly PS-compatible, painting-biased | historically divergent; PS scheme opt-in | **PS scheme, hard-coded** |
| Menu geography | — | close-ish | notably different (Colors menu, Script-Fu) | **PS-shaped** (Image/Layer/Select/Filter) |
| Single-window UI | yes | yes | now (was infamously not) | **yes, native** |
| Non-destructive adjustment layers | yes | partial | weak | **yes** |
| 16-bit / raw | yes | yes | yes (2.10+) | **no — 8-bit sRGB core** |
| Onboarding for non-technical | medium | medium (painter-first) | **poor** ("it's not Photoshop") | medium, PS-familiar |

Lessons: **GIMP** is the cautionary tale — capable and free, repeatedly
rejected on interface unfamiliarity and a late single-window fix. **Krita** is
the encouraging one — it won digital painters by nailing *their* muscle memory
and not trying to be everything. C-Shop already made the interface choice both
imply: it is Photoshop-shaped by default.

## 10. The muscle-memory problem

For an existing user, **the shortcuts and menu geography are the product** more
than the feature list — which is exactly why editors lose users to "it's not
Photoshop". C-Shop's `shortcuts.rs` and menu bar are deliberately PS-shaped
(Ctrl+J "Layer via Copy"; V/M/L/W/B/S/E/G/T/P tool letters;
Image/Layer/Select/Filter menus). That is a head start a new project would
re-derive and a GIMP-style base would fight.

Your audience is **neither PS pros nor painters — it is AI-first creators**, so
you inherit *less* muscle-memory debt than either: PS-shaped defaults are
"familiar enough" without being a feature-parity promise you must keep.

## 11. Fit for the AI-assisted workflow

The AI integration is this project's strongest fit because the plumbing exists:

- `cshop-app/src/mcp/` is a **sandboxed, session-holding server** — one document
  per session on a single owner thread — exposing six tools, with `place`
  (PNG → layer), `export` (region → file), selections-as-masks, and
  **image-carrying results** (describe → draw → look → correct over the wire).
- This *is* the img2img loop. An external orchestrator can drive SDNext/ComfyUI
  today with zero editor changes (Option A).

Scorecard for the **non-technical, AI-first** audience (1–5):

| Need | Fit | Note |
|---|---|---|
| txt2img creation | 5 | server + place-as-layer already there |
| img2img / inpaint enhance | 5 | export-region + masks + place = the loop, built |
| Basic masks | 5 | Quick Mask, selection→mask, channels — tested |
| Basic adjustments | 5 | Levels/Curves/etc; non-destructive layers |
| Familiar UI | 4 | PS-shaped already; needs simplifying |
| Non-technical onboarding | 3 | PS-shaped ≠ beginner-simple; needs a trimmed mode |
| Raw photographer workflow | 5* | *delegated to RapidRAW (§13); C-Shop owns compositing/enhance |

An **in-app Generate/Enhance panel** (Option B) needs four small pieces that do
not exist yet: a GUI async task layer, an HTTP client, a config store, and
layer provenance. After Phase 1 it lands as one command + one tool.

## 12. The raw / photographer caveat

The one real limitation, and it is a **core-model** limit, not a UI one:

- Pixels are **8-bit sRGB throughout** (`cshop-core/src/pixels.rs`: "8-bit
  straight-alpha sRGB"; `PixelBuffer` is `Vec<Rgba8>`). Filters compute in f32
  internally but round-trip back to 8-bit.
- There is **no raw decoder** in the tree, and PSD import already refuses 16-bit
  and CMYK.
- Therefore **opening a .CR2/.NEF/.ARW and developing it is not supported and
  not cheap** — true raw implies a 16-bit/float pipeline touching `PixelBuffer`,
  the compositor and every filter.

What works cheaply and honestly: accept a raw **demosaiced to 8-bit** (TIFF/
PNG/JPEG), then mask → adjust → SD-enhance. Since SD img2img is itself an 8-bit
round trip, the distinction is invisible to a non-technical user.

**Recommendation:** do not attempt a 16-bit/raw core rewrite. **Delegate raw
development to RapidRAW** (next section); C-Shop stays "photo editing + AI
enhancement" and gains the raw front-end for free by handoff.

## 13. RapidRAW: delegating the raw workflow

Rather than absorb the 16-bit/raw cost, pair C-Shop with **RapidRAW**
(CyberTimon/RapidRAW) — a mature, actively developed Lightroom-style raw
developer. The two are **complementary, not competing**, and the pairing
retires the only weak row in the scorecard.

**What RapidRAW is** (verified against its repo/release notes): Rust + Tauri +
React, a **32-bit** WGSL GPU pipeline, full raw decode via **rawler/dnglab**
(Canon/Nikon/Sony/Fuji/DNG…), non-destructive `.rrdata` sidecars, tone/colour/
HSL/curves/LUTs, Lensfun correction, **AI masks** (SAM 2 subject, U-2-Net sky/
foreground, Depth Anything v2), local LaMa inpaint, and **ComfyUI** generative
edits via a separate "RapidRAW-AI-Connector" middleware. It is licensed
**AGPL-3.0**.

**Why it fits as the upstream stage:** RapidRAW has **no layers, type, shapes,
or arbitrary compositing** — exactly what C-Shop provides. The seam is the
*developed 8-bit image*: RapidRAW develops the raw (the high-bit-depth work
happens in the tool built for it), then hands a rendered frame to C-Shop for
compositing and agentic/generative editing. Nothing is lost at the boundary
because SD img2img is itself an 8-bit round trip.

**Two front doors, one handoff.** Users pick by intent: *"making something"* →
C-Shop (blank canvas, place generated layers, type/shapes); *"fixing a photo"*
→ RapidRAW (develop, ComfyUI-enhance), coming to C-Shop only when compositing
is needed.

**Two hard constraints:**

1. **Licensing.** C-Shop is MIT/Apache-2.0; RapidRAW and its connector are
   **AGPL-3.0**. Do **not** copy RapidRAW code into C-Shop — it would force
   C-Shop to AGPL and could trip the network clause on the served editor.
   Integration must be **arm's-length** (separate processes, file handoff, or
   both talking to ComfyUI independently), which is clean under AGPL.
2. **Don't reinvent the ComfyUI glue.** When C-Shop builds its own Generate/
   Enhance panel, **mirror RapidRAW's middleware *architecture*** (a local
   sidecar owning workflow injection + image caching, spoken to over HTTP) —
   copying the *design*, not the *code* — which fits the `cshop-net` plan and
   keeps ComfyUI complexity out of the editor process.

**Integration, cheapest first:**
- **A. "Open With" file handoff (zero code).** RapidRAW already ships external-
  editor support; register C-Shop, and it opens RapidRAW's exported TIFF/PNG.
- **B. Shared workspace.** Point RapidRAW's export dir at C-Shop's `--serve`
  workspace; a C-Shop MCP session runs `place out.png` into the agentic loop.
- **C. Shared ComfyUI (later).** C-Shop's own panel talks to the same ComfyUI
  server independently, using RapidRAW's connector only as a design reference.

## 14. The Liquify tool: a near-term priority

Some corrections need a human eye, not a diffusion model — reshaping, slimming,
nudging a feature into place. **Liquify is the one push-pixels tool the target
audience will expect, and it fits this codebase unusually well.** It should be
an early plugin, not a "someday".

**Why it fits.** Liquify is a *forward-warp brush*: the user drags, and pixels
follow. The core already contains the exact machinery:

- `cshop-core/src/filters/distort.rs` implements every distortion as a
  **backward map** over an f32 `Plane` — "for each destination pixel, work out
  where it came from and sample there" — parallelised with rayon and sampled
  with bilinear `Plane::sample`. Twirl, Pinch and Spherize are already
  brush-like radial warps in closed form.
- The filter dialogs already **preview on a downscaled proxy** and commit at
  full resolution, which is precisely the interaction Liquify needs to stay
  responsive on large images.

So Liquify is not new infrastructure — it is a *displacement field* driving the
existing `warp()`, where the field is accumulated from brush strokes instead of
computed from a formula. The standard Liquify sub-tools (Forward Warp, Pucker,
Bloat, Twirl, Push-Left) are each a small vector-field kernel stamped under the
brush; Pucker/Bloat/Twirl are literally the radial maths already in `distort.rs`.

**How it lands in the modular plan.** After the Phase 3 `ToolRegistry`, Liquify
is a self-contained plugin:

- **State** (the accumulated displacement field, brush size/pressure/mode)
  lives on the tool struct — exactly the god-object-avoidance the plan is built
  for. No `CShopApp` fields.
- **Core** gains a `liquify` module beside `distort.rs`: a `DisplacementField`
  (two `f32` channels sized to the layer, or a coarser mesh interpolated up) and
  an `apply(plane, field)` that reuses the existing backward-map/`sample` path.
- **Preview** uses the same proxy trick; **commit** is one undoable step through
  the history system that already snapshots pixels.
- **Non-destructive option (later):** because a warp is fully described by its
  field, Liquify *could* be stored as a re-editable layer modifier and absorbed
  by the tagged-chunk `.cshop` format — the same way effects are. Ship the
  destructive version first; keep the field serialisable so this stays open.

**Cost and risk.** Small-to-medium and low-risk: it reuses the warp/sample/
proxy/history seams rather than adding any. The only genuinely new piece is the
brush→field accumulation (a Gaussian-weighted vector stamp), which is
self-contained and unit-testable on the CPU against a known field, in the same
CPU-reference spirit as the rest of the core. It needs no GPU work to be
correct (the distort filters are CPU), though it can move to a shader later if
interactive latency on huge layers demands it.

**Verdict:** Liquify is the highest-value *human-eye* tool for this audience and
one of the cheapest to add, because the hard parts (backward-map warp, bilinear
sampling, proxy preview, undo) already exist and are tested. Schedule it as the
first non-trivial tool plugin after the registry lands.

## 15. Cloud and mobile: a future infrastructure track

Modern Lightroom's pull is not just the desktop app — it is **sync + a mobile
app that edits the same catalog**. For a non-technical, AI-first audience that
expectation is real, so it must be *kept in mind* even though it is **not a
near-term priority**. Treat it as a separate infrastructure track, not a fork
feature, and make today's decisions compatible with it rather than building it
now.

**What already points the right way.** The project is unusually well-placed for
a server story because it was built headless-first:

- `cshop-app` already runs **without a window**, over MCP, with **sandboxed,
  session-holding** documents on a single owner thread (`mcp/editor.rs`), a
  workspace confined to one directory, a loopback/token security model, and
  **image-carrying results**. That is most of the skeleton of a render/edit
  service.
- Edits are already expressible as **scripts** and **`Action`s** — a compact,
  serialisable representation of "what was done", which is exactly what a
  sync/replay model wants instead of shipping full pixels.
- The `.cshop` format is **forward-compatible** (tagged chunks), so documents
  authored by a future mobile client that a desktop build predates will still
  open.

**What it would require (deliberately deferred).** This is genuinely an
infrastructure project with its own risk surface, and it introduces the
"complex Stable Diffusion workflow" the user already flagged as out of scope for
now:

- **Storage + identity + sync.** A document store, accounts, and a
  conflict/merge model. Non-trivial; this is the bulk of the work and unrelated
  to image editing.
- **A thin mobile client.** Realistically not the Rust/egui desktop UI recompiled,
  but a lightweight client (touch-first) that renders server-composited results
  and sends back `Action`s/scripts. The server does the GPU work; the phone
  shows pictures and captures intent — which is exactly the shape the MCP
  server already has.
- **Server-side GPU or software render.** The repo already ships a **software
  Vulkan** path (the Docker image renders with no GPU), so a cloud renderer is
  not a new capability — it is a scaling and cost question.
- **Multi-user hardening.** `panic = "abort"` (see Risks) becomes unacceptable
  once one process serves many users; and the single-owner-thread session model
  would need a pool.

**Guidance for now (cheap, keeps the door open):**
1. **Keep edits representable as data** (`Action`/script), not just as pixels —
   the modular plan already reinforces this. Sync later replays data.
2. **Keep the `.cshop` format forward-compatible** — already true; don't break it.
3. **When building the `cshop-net` crate and task runner** (already on the
   roadmap for SD), design them so the *same* client/transport could talk to a
   remote render service, not only a localhost ComfyUI. Same abstraction,
   different endpoint.
4. **Do not** stand up accounts, storage, or a mobile client until the desktop
   fork is mature; premature infra would starve the editor work that is the
   actual product.

**Verdict:** a real and audience-appropriate goal, but an infrastructure track
to *design toward, not build now*. The headless server, the data-shaped edit
model, and the software renderer mean the fork is already leaning in the right
direction; the job for now is to avoid decisions that would foreclose it.

## 16. Build vs. rebuild decision

**Build on C-Shop.** A rewrite discards ~38k lines of tested Rust — compositor,
27 cross-checked blend modes, `.cshop` + PSD codecs, and the entire MCP/script
harness that *is* the AI surface — to avoid a UI that is already PS-shaped
(what you want) and whose real problems (internal structure, scope) are fixable
in place via the bounded, phased modular plan. Rebuilding trades a *planned,
bounded* refactor for an *unplanned, unbounded* reimplementation of the parts
that are already the best in the repo.

## 17. What to keep, trim, and add

**Keep:** `cshop-core` entirely; `cshop-gpu`/`cshop-io`; the `Action`/`run()`
seam; styles, script, MCP; and the cross-check/round-trip/fuzz test suites (the
refactor's safety net).

**Trim** (by un-registering plugins, not deleting code): most of the 30 filters
(SD replaces stylisation); Pen/Shape/vector layers, advanced type and layer
effects if you want genuinely "basic".

**Add** (each small, per the assessment):
1. A `cshop-net` crate — extract the existing MCP HTTP/JSON, add a client.
2. A **GUI task runner** — none exists (no `thread::spawn` in `cshop-ui`,
   verified) — so a ~20s generation doesn't freeze the window.
3. A **config store** + **layer provenance** ("this layer came from this
   prompt/model/seed").
4. A **Generate/Enhance panel** on top — one command + one tool after Phase 1.

## 18. Risks

- **The refactor must actually happen.** If phases 1–3 slip, you pile AI
  features onto the god object. Gate new AI UI behind Phase 1 landing.
- **Async is genuinely absent in the UI** (verified). Build the task runner
  *before* any Generate panel; a call on the synchronous loop freezes the
  window.
- **`panic = "abort"`** (release profile) means a bad network response can kill
  the app; wrap the client and revisit the profile before shipping.
- **UI tests need a GPU and there is no CI.** Stand up CI (GPU-less for
  core/io/net at least) before growing the surface.
- **`EditorCtx` is load-bearing.** If it leaks `&mut CShopApp`, the boundary is
  fake; keep it minimal and grow only when a real plugin needs a capability.

## 19. Recommended order of work

1. **Bridge SD over MCP** (Option A) — zero editor changes; proves the workflow
   and exercises the sandbox from outside.
2. **Wire the RapidRAW handoff** (§13, "Open With" + shared workspace) — near-
   zero code, delivers the raw/photographer flow immediately.
3. **Phase 0 + Phase 1** (command registry) on a branch — UI-only, no behaviour
   change, no GPU needed; makes "add a menu entry" one line.
4. **`cshop-net` crate** — extract HTTP/JSON, add a client; design the transport
   so it could also reach a remote render service later (§15), not only ComfyUI.
5. **GUI task runner** with document identity and loop wakeup — the gap
   everything else falls into.
6. **Config store + layer provenance.**
7. **Generate/Enhance panel** as a plugin (mirror RapidRAW's ComfyUI-middleware
   architecture, not its AGPL code).
8. **Phase 2 (effects) and Phase 3 (tools)** as appetite allows; **Phase 4**
   trims scope to taste.
9. **Liquify plugin** (§14) — the first non-trivial tool after the registry;
   reuses the existing warp/sample/proxy/history seams.
10. **Cloud/mobile track** (§15) — design toward it, build it only once the
    desktop fork is mature; keep edits data-shaped and the format compatible.
11. **Do not pursue dynamic plugins.** Compile-time exhaustiveness is a feature
    here; a registered-into-registries crate is the idiomatic extension path.

## 20. Appendix: verified facts and citations

Checked in source on this tree:

- **8-bit sRGB core.** `cshop-core/src/pixels.rs` — module doc "Layer pixels
  live here as 8-bit straight-alpha sRGB"; `PixelBuffer { data: Vec<Rgba8> }`;
  `Rgba8 { r: u8, g: u8, b: u8, a: u8 }` in `color.rs`. Filters use an f32
  `Plane` internally (`filters/plane.rs`) but convert back to 8-bit.
- **No raw decoder / no HTTP client / no async in UI.** A search for
  `thread::spawn|tokio|reqwest|async fn|rawloader|dcraw|txt2img|img2img` across
  the crates matched only: `cshop-gpu/src/context.rs`, `cshop-core/src/font.rs`,
  `cshop-app/src/mcp/http.rs`, and `cshop-app/tests/mcp.rs` — i.e. no raw, no
  client library, and no `thread::spawn` anywhere in `cshop-ui`.
- **MCP server shape.** `cshop-app/src/mcp/editor.rs` — one document per session
  on a single owner thread, sessions time out (30 min) and cap (32);
  `mcp/tools.rs` — six tools with image-carrying results.
- **Command pattern.** `cshop-ui/src/commands.rs` — `Action` enum;
  `cshop-ui/src/app.rs:737` — the single `run(&mut self, action)` match.
- **Closed `Tool` enum + TOOL_GROUPS.** `cshop-ui/src/tools.rs` — 19-variant
  enum with parallel metadata matches; toolbar/shortcut coupling in `chrome.rs`,
  `shortcuts.rs`, `app.rs`, `canvas.rs`.
- **Filter catalog (the good pattern).** `cshop-core/src/filters/mod.rs` —
  `Filter::all_defaults()` drives the Filter menu generically;
  `adjust.rs` has a parallel `all_defaults()`.
- **Warp machinery (Liquify substrate).** `cshop-core/src/filters/distort.rs` —
  `warp()` is a rayon-parallel **backward map** over an f32 `Plane`;
  `Plane::sample` does bilinear interpolation (`filters/plane.rs`); Twirl/Pinch/
  Spherize are radial warps in closed form. Filter dialogs preview on a
  downscaled proxy and commit at full resolution (`filters/mod.rs`).
- **Build state.** `cargo check --workspace --tests` passes; core/io tests
  pass; UI tests require a GPU (segfault headless); no CI config present.

Checked against external sources:

- **RapidRAW.** CyberTimon/RapidRAW README + release notes (retrieved
  2026-08-30): Rust/Tauri/React, 32-bit WGSL pipeline, raw via rawler/dnglab,
  `.rrdata` sidecars, AI masks (SAM 2 / U-2-Net / Depth Anything v2), LaMa
  inpaint, ComfyUI via the separate "RapidRAW-AI-Connector"; **no layers/type/
  compositing**; licensed **AGPL-3.0**; ships "Open With" external-editor
  support and a headless export CLI.
```