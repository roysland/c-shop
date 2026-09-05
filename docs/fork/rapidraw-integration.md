# RapidRAW and the raw workflow: division of labour

A follow-up to [`build-vs-rebuild.md`](build-vs-rebuild.md), which found the one
real limitation of C-Shop for the target workflow was raw photography — an
8-bit sRGB core with no raw decoder. This note evaluates the proposal to **let
a dedicated app (RapidRAW) own the raw/develop stage** and keep C-Shop for the
compositing/AI-editing stage.

Verified against RapidRAW's own repository and release notes (CyberTimon/
RapidRAW, ~9.7k stars, active daily development as of 2026-08). Facts cited
inline.

## Short answer

**Yes — this is the right architecture, and it removes the one weak score from
the earlier assessment.** RapidRAW and C-Shop are genuinely complementary, not
competitors: RapidRAW is a Lightroom-style *non-destructive raw developer*
(catalog, sliders, masks, per-image sidecars), and C-Shop is a *compositing
editor* (layers, type, shapes, blend modes, the AI place/export loop). The
photographer flow becomes a two-app pipeline, each doing what it is actually
built for.

**But two facts change the integration design and must be decided up front:**

1. **RapidRAW is AGPL-3.0.** C-Shop is MIT OR Apache-2.0. You cannot copy
   RapidRAW code into C-Shop without relicensing C-Shop to AGPL. Integration
   must therefore be **arm's-length** (separate processes, files, or a network
   boundary) — which, happily, is exactly how both already prefer to work.
2. **RapidRAW is not a dedicated image editor and does not try to be.** It has
   no layers, type, shapes, or arbitrary compositing. That gap is precisely
   what C-Shop fills, so the two do not overlap — they hand off.

## What RapidRAW is (verified)

- **Stack:** Rust + Tauri + React frontend, GPU pipeline in WGSL, **full 32-bit
  processing**. So it already solves the high-bit-depth raw pipeline that would
  be a major core rewrite in C-Shop.
- **Raw:** full raw decode via **rawler/dnglab** (Canon, Nikon, Sony, Fuji
  X-Trans, DNG, etc.), plus JPEG/TIFF/PNG/WebP/JXL/EXR/HDR and more.
- **Editing model:** *non-destructive*, all edits in a `.rrdata` sidecar; the
  original file is never touched. Adjustments, tone curves, HSL, color grading,
  LUTs, lens correction (Lensfun), detail/denoise, transform/crop.
- **Masking:** brush, linear, radial, parametric colour/luminance, and
  **AI masks** — subject (SAM 2), sky/foreground (U-2-Net), depth (Depth
  Anything V2) — running locally.
- **AI / generative:** three tiers — built-in local (LaMa inpaint, CLIP tagging,
  AI masks), **self-hosted ComfyUI via a separate middleware ("RapidRAW-AI-
  Connector")**, and a planned cloud service. The ComfyUI path sends the image
  once then only masks+prompts per edit.
- **Library:** folders, culling, ratings, tags, virtual copies, batch export,
  a headless export CLI, camera tethering.
- **Licence:** **AGPL-3.0**, chosen deliberately to keep derivatives open.

## Where the two apps divide cleanly

| Stage | Owner | Why |
|---|---|---|
| Ingest raw, develop, cull, catalog | **RapidRAW** | 32-bit raw pipeline + library already built; C-Shop has neither |
| Tonal/colour grade, lens correction | **RapidRAW** | its core competence; scene-referred, high-bit-depth |
| AI masks (subject/sky/depth) | **RapidRAW** | ships local models; C-Shop has none |
| Export a developed 8-bit image | **RapidRAW** → file | "Open With" / export to TIFF/PNG |
| Layered compositing, type, shapes | **C-Shop** | RapidRAW has no layers at all |
| Blend modes, layer effects, masks-as-layers | **C-Shop** | its core competence |
| Scripted/agentic + MCP AI editing loop | **C-Shop** | the sandboxed server + place/export loop |
| txt2img from scratch onto a canvas | **C-Shop** | RapidRAW is photo-first; it enhances an existing frame |

The seam is the **developed 8-bit image**: RapidRAW hands off a rendered frame,
C-Shop composites and does agentic/generative work on it. This is the same
8-bit boundary the earlier report identified — except now the high-bit-depth
work happens *upstream in the tool built for it*, so nothing is lost.

## What this does to the earlier verdict

The single weak row in the fit scorecard was:

> Raw photographer workflow — **2/5** — 8-bit core; "enhance" yes, "develop" no.

Delegating develop to RapidRAW **retires that row entirely.** C-Shop no longer
needs — and should not attempt — a 16-bit/raw core rewrite. The build-on-C-Shop
decision gets *stronger*, and the scope gets *smaller*: the fork is now
unambiguously "a basic modular compositing + AI editor", and raw is Someone
Else's Program.

It also reframes txt2img vs. photo-enhance:

- **txt2img-first creation** → C-Shop is the home (blank canvas, place
  generated layers, composite, type/shapes).
- **photo-first enhancement** → RapidRAW is the front door (develop the raw,
  ComfyUI-enhance in place), and only *comes to C-Shop when the user needs
  compositing* (multiple images, text, graphic layers) that RapidRAW cannot do.

Two front doors, one handoff. Non-technical users pick by intent ("I'm making
something" vs. "I'm fixing a photo"), not by learning two toolboxes at once.

## Integration options, cheapest first

### A. File handoff via "Open With" (ship this first; zero code)
RapidRAW already has **"Open With" external-editor support** (release note
2026-07-03). Register C-Shop as an external editor; RapidRAW exports a
developed TIFF/PNG and opens it in C-Shop. C-Shop already opens those formats.
**Effort: configuration only. Licence: clean — two separate programs.**

### B. Shared workspace folder (light glue)
Point RapidRAW's export directory at C-Shop's `--serve` workspace. RapidRAW
develops → writes a PNG → a C-Shop MCP session runs `place out.png`. The
agentic loop then composites/enhances further. **Effort: a few lines of config
+ an orchestrator script. Licence: clean — file boundary.**

### C. Adopt RapidRAW's ComfyUI middleware pattern (don't rebuild it)
When C-Shop grows its own in-app Generate/Enhance panel (Option B of the
assessment), **mirror RapidRAW's "AI Connector" design** rather than inventing
one: a small local middleware that owns ComfyUI workflow injection and image
caching, talked to over HTTP. This fits C-Shop's existing `cshop-net`-extraction
plan and its "no heavyweight deps in-tree" constraint, because the ComfyUI
complexity lives in the sidecar process. **Do not link or copy RapidRAW's
connector** (AGPL); use it as a *design reference* and speak the same ComfyUI
API. Licence: clean if independently implemented against ComfyUI's public API.

## Licensing: the one hard constraint

- C-Shop: **MIT OR Apache-2.0** (permissive).
- RapidRAW and its AI-Connector: **AGPL-3.0** (strong copyleft, network clause).

Consequences the fork must respect:
- **Do not copy RapidRAW/Connector source into C-Shop.** It would force C-Shop
  to AGPL and could trip the network clause for the served editor.
- **Arm's-length interop is fine:** separate processes, file handoff, or
  talking to the same ComfyUI server independently. AGPL governs *its* code,
  not files it produces or a program that merely launches it.
- If you ever *want* the AGPL, that is a deliberate project-wide choice, not a
  side effect — decide it explicitly, don't drift into it.

## Recommendation

1. **Adopt the two-app model.** Drop raw/develop from C-Shop's scope for good;
   position RapidRAW as the upstream developer and C-Shop as the downstream
   compositing + agentic-AI editor. This simplifies C-Shop and plays to both
   tools' strengths.
2. **Ship integration A (Open With) immediately** — it is free and proves the
   pipeline end to end.
3. **Use integration B** to connect RapidRAW output to C-Shop's MCP `place`
   loop for the agentic path.
4. **When building C-Shop's Generate panel, copy RapidRAW's *architecture*
   (middleware-per-ComfyUI), not its code**, and keep the AGPL boundary at the
   process edge.
5. **Update the docs** so the fork's stated scope reads: *a basic modular
   compositing and AI-assisted image editor; raw development is delegated to a
   dedicated tool.*

This is the cleaner product story than either app alone: RapidRAW answers "it's
not Lightroom", C-Shop answers "it's not Photoshop", and the AI generation loop
is the thread that runs through both.
```