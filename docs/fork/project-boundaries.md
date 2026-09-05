# Two projects, one handoff: boundaries between the UIs

This note solidifies the boundary between the two products discussed across
`docs/fork/`. It supersedes the looser "how the products relate" sketches in
[`beginner-first-direction.md`](beginner-first-direction.md) and the
[`README.md`](README.md) by stating the boundary as a contract, not a diagram.

The conclusion reached with the user: **these are two completely separate
projects for two different people, and they converge at exactly one point — an
import/export handoff of a rendered image (plus optional recipe metadata) across
a process boundary.** No shared codebase, no shared data model, no shared UI.

---

## The two projects, stated as a boundary

### C-Shop — the Photoshop clone that Krita and GIMP don't deliver

**Owner/user:** the technical user (procedural, control-seeking, Linux-native).

**What it is:** a full, procedural, layer-and-tool image editor with the
Photoshop *interaction grammar* — pick a tool, act on the canvas, tune options,
manage layers, masks, and blend modes. It is Photoshop-shaped **on purpose**.

**Why it exists (the gap it fills):** Krita and GIMP both fail the
Photoshop-refugee for different reasons —
- **GIMP:** divergent shortcuts and menu geography, a single-window story it
  fixed late, and an onboarding reputation defined by "it's not Photoshop."
- **Krita:** excellent, but painter-first — its defaults, dockers, and mental
  model are tuned for digital painting, not photo compositing/retouching.

C-Shop already made the choice both avoid: a hard-coded **PS shortcut scheme**
(Ctrl+J "Layer via Copy", the V/M/L/W/B/S/E/G/T/P tool letters) and **PS-shaped
menus** (Image / Layer / Select / Filter), single-window native, with
non-destructive adjustment layers. For someone with Photoshop muscle memory on
Linux, *that familiarity is the product* — more than any single feature. This is
the niche neither Krita nor GIMP resolves cleanly, and it is what C-Shop is for.

**Interaction grammar:** procedural. Control over every step is the whole value.

**Engine:** its own — the tested Vulkan compositor, layered `.cshop`/PSD codecs,
and the MCP/script harness. Composites locally.

### The Look Editor — a minimal prompt/comment enhancement UI

**Owner/user:** the artistic, non-technical user (intuitive, outcome-seeking; a
domain expert in makeup whose ADHD/dyslexia sharpen a fast visual intuition).

**What it is:** a **minimal** UI whose entire job is to enhance an image through
**comments/prompts and pointing** — "warm the light", "deepen the lip", "soften
the skin" — with no tools, layers, or blend modes ever shown. She expresses
judgment; the machine does the procedure. (This is the already-built Visual Look
Editor prototype in `~/Projects/ai-image`.)

**Why it exists (the gap it fills):** a procedural editor forces the user to
translate a *seen* result into a *sequence of tool operations*. That translation
is the tax, and it is exactly the step her strengths let her skip. A minimal
prompt-driven UI removes the translation instead of the power.

**Interaction grammar:** intuitive/outcome-first — pick an outcome → point at
where → see the result. Never "select a tool."

**Engine:** ComfyUI. The "Visual Locks" (keep pose / face / light) map onto
ComfyUI conditioners (ControlNet / PuLID / IC-Light), which C-Shop cannot host.
Generation belongs to ComfyUI, not C-Shop.

---

## Why they must stay separate (not one program with two modes)

The two products have **opposite interaction grammars**, and the grammar — not
the feature count — is what fits or excludes a user:

| | C-Shop | Look Editor |
|---|---|---|
| Grammar | tool → act → tune (procedural) | outcome → point → see (intuitive) |
| Unit of work | a tool operation on a layer | a comment/prompt on a region |
| "Control of every step" | the point / a feature | the burden / an anti-feature |
| Layers, masks, blend modes | first-class, visible | never shown |
| Engine | own compositor (local) | ComfyUI (conditioners) |
| Frontend | native egui, PS-shaped | minimal web, prompt-driven |

You **cannot** turn one into the other by hiding menus. A trimmed C-Shop is a
crippled Photoshop — same grammar, still excludes her. A Look Editor with more
knobs is a worse prompt UI — it re-introduces the translation tax. The same
feature ("expose every step") is a feature for one user and an anti-feature for
the other. That asymmetry is the proof they are two projects.

---

## The single convergence point: an image handoff

They meet at **one seam only**: a rendered image crosses a process boundary,
optionally carrying recipe/metadata. Everything else stays independent.

```
   ┌───────────────────────┐        ┌───────────────────────┐
   │   Look Editor (web)    │        │   C-Shop (native)     │
   │  minimal prompt UI     │        │  PS-clone editor      │
   └──────────┬────────────┘        └───────────▲───────────┘
              │ generate                          │ import (place)
              ▼                                    │
        ┌───────────┐                              │
        │  ComfyUI  │──── rendered image ──────────┘
        └───────────┘         (+ optional recipe metadata)

   ── the ONLY link between the two products is this image handoff ──
```

**The contract is deliberately tiny:**
- **What crosses:** a PNG/TIFF (the rendered pixels), and *optionally* a small
  JSON recipe/metadata sidecar (prompt, region, seed, model — provenance).
- **What does NOT cross:** no shared library, no shared data model, no `.cshop`
  document, no layer stack, no code. Neither project depends on the other's
  types or build.
- **Direction:** primarily Look Editor/ComfyUI → C-Shop ("edit further" — bring
  a generated result into the real editor to composite/retouch). A reverse path
  (C-Shop exports a flat image the Look Editor ingests) is possible but not
  required.

**Why this is cheap: C-Shop's receiving end already exists.** The MCP server's
`place` command ingests a PNG as a real layer, and `export` writes a region back
out. So the file-level handoff works **today** with zero new C-Shop code; a
richer MCP/SDK handoff (carry the recipe, drive a session) comes for free once
the `cshop-net` crate and task runner land on the existing roadmap.

---

## Boundary rules (the contract, explicitly)

To keep the seam from eroding into coupling:

1. **File first, MCP/SDK later.** v1 is literally "write a PNG, open it in the
   other app." Do not build a shared protocol before the file handoff has
   proven the workflow.
2. **The image is the interface.** Anything richer than an image + a flat JSON
   sidecar is a smell. If a feature needs the two apps to share a document
   model, it belongs inside one app, not across the seam.
3. **No code sharing across the boundary.** Not the egui UI, not the `Tool`/
   `Action` grammar, not the Look Editor's web components. Patterns may be
   independently re-derived (e.g. a catalog/registry shape appears in both), but
   that is convergent design, not a shared dependency.
4. **Both apps are ComfyUI clients, each sized to its resolution regime.** The
   Look Editor sends whole ~1024 images; C-Shop sends *downscaled native-res
   regions* for its own object-removal feature (see
   [`magic-eraser.md`](magic-eraser.md)). C-Shop never tries to host the Visual
   Locks (they are ComfyUI conditioners the Look Editor owns); the Look Editor
   never tries to grow a layer stack. Neither runs diffusion over a whole 60MP
   frame — C-Shop avoids that by scaling the *region* to the engine, not the
   engine to the photo.
5. **Metadata is additive and optional.** A recipe sidecar enriches the handoff
   but is never required for it to work — the image alone must always be a valid
   handoff.

---

## What each project owns (no overlap)

| Concern | Owner |
|---|---|
| PS-familiar shortcuts, menus, tools | **C-Shop** |
| Layers, masks, blend modes, type, shapes | **C-Shop** |
| Local compositor + `.cshop`/PSD formats | **C-Shop** |
| Web/social delivery (her render is final) | **Look Editor / her export** (Tier 1 — see [`delivery-model.md`](delivery-model.md)) |
| Print/agency master (fresh full-res pass, overwrites her export) | **C-Shop** (Tier 2, gated by her taste) |
| Minimal prompt/comment enhancement UI | **Look Editor** |
| Visual Locks, swatches, branching, Recipe Card | **Look Editor** |
| Prompt → conditioner compilation | **Look Editor** |
| txt2img / img2img / inpaint generation | **ComfyUI** (shared by both apps) |
| Whole-image generation, Visual Locks, recipes | **Look Editor** (its ComfyUI client) |
| Object removal at photo resolution (magic eraser) | **C-Shop** (its own ComfyUI client, region round-trip) |
| The image (+ optional recipe) handoff | **the seam** — file, then MCP/SDK |

---

## Honest consequences

- **C-Shop delivers little to *her* daily workflow** beyond being an "edit
  further" target and the home of the delivery/export screen. That is fine — the
  PS-clone earns its keep with *his* audience (him, prosumers, Photoshop
  refugees on Linux). Do not justify the C-Shop fork on the Look Editor's behalf.
- **Two products plus ComfyUI is real maintenance surface.** The file/HTTP-only
  boundary is exactly what keeps that manageable — each can be built, debugged,
  released, and owned alone.
- **The seam is small on purpose.** Resisting the urge to "integrate more
  tightly" is the design discipline that keeps both projects simple. Tighter
  coupling would buy a slightly smoother demo at the cost of two entangled
  codebases — the opposite of the goal.

## Bottom line

Two separate projects, two users, opposite grammars, one tiny contract:

- **C-Shop** is the Photoshop clone Krita and GIMP don't deliver — procedural,
  PS-shaped, native, *his*.
- **The Look Editor** is a minimal prompt/comment enhancement UI over ComfyUI —
  intuitive, outcome-first, *hers*.
- They converge at **a single image (+ optional recipe) handoff**, and C-Shop's
  existing MCP `place`/`export` is already most of the receiving half. No shared
  code, no shared model, no UI convergence — just import/export.
