# The magic eraser: SD object removal at photography resolution

A follow-up to [`project-boundaries.md`](project-boundaries.md). Where that note
set the boundary between the two products, this one pins down the *one* Stable
Diffusion feature C-Shop must have and why it is both a hard requirement and a
bounded one. Verified against the tree (facts cited inline).

## Why SD in C-Shop is different from SD in the Look Editor

The two products live in different **resolution regimes**, and the regime — not
the task — decides whether SD is the easy path or the hard one.

- **Look Editor / ComfyUI — generation resolution.** Images are *born* at
  diffusion-native size (~1024, SDXL tiles). The working size *is* the model
  size, so "remove a stray hair" is trivially "inpaint the region, mask a
  natural stop." SD is the easy, primary engine there.
- **C-Shop — native photography resolution.** Images are 50–60MP raws turned to
  8-bit — 7000–9000px on the long edge, ~40× the linear resolution of a 1024
  tile. Diffusion models do not run natively at that size; running SD on the
  whole frame is impossible or produces garbage, and this is why the classic
  full-resolution, resample-free tools (healing brush, clone, content-aware
  fill) exist in the first place.

So the AI-does-everything story that is true for the Look Editor is *false* for
C-Shop retouching — not because the tasks differ but because the resolution
regime makes whole-image SD the hard path.

## The essential primitive: scale the work to the engine

The user's actual retouching need is **removal of unwanted objects** on a large
photo, and the mechanism that makes SD tractable at 50MP is to **scale the
region down to SD's home turf**, not to scale SD up to the photo:

1. **Mark** a region around the unwanted object on the full-resolution image.
2. **Downscale** that region (with padding/context around the mask) to an
   SD-workable size (~1024).
3. **Inpaint/remove** in SD at that comfortable resolution, via the external
   engine (ComfyUI).
4. **Upscale** the returned region back to the region's exact native pixel
   dimensions.
5. **Composite** it back into the full-res image through a feathered mask
   ("natural stops"), as one undoable step.

The reason this works — and why it is the *reliable* SD use rather than the
risky one — is that **object removal tolerates the softness that the
downscale→upscale round-trip introduces.** A removed object becomes plausible
background/skin/fabric; it does not need to carry 60MP micro-detail the way a
*sharpened subject* would. The same round-trip that would ruin a face is exactly
right for erasing something. C-Shop never asks SD to work at full resolution, so
the hard high-res-SD problem (tiling a whole 60MP image, seam-matching across a
huge frame) is *avoided by construction*, not solved.

This is the "eraser magic": a first-class C-Shop tool, bounded to
region-round-trip inpainting, not "run SD on the photo."

## What already exists (verified in source)

The imaging core of this pipeline is present and battle-tested — the hard parts
are already built:

- **High-quality arbitrary-dimension resize**, premultiplied and area-aware,
  with Lanczos-3 for reduction — `cshop-core/src/resample.rs` `resize(src, w, h,
  filter)`. This is both the downscale (step 2) and the upscale (step 4) in one
  call each.
- **Feathered selections and full mask algebra** — `cshop-core/src/selection.rs`
  (`feather`, `expand`/`contract`/`border`/`smooth`, `bounds()`, `to_mask()`)
  and `cshop-core/src/mask.rs` (`copy_rect`, `paste`, `coverage_bounds`). The
  feathered selection *is* the natural-stop mask.
- **Magic Wand** region selection with tolerance/contiguity —
  `cshop-core/src/wand.rs` — a natural-stop *region selector*.
- **Place a PNG as a positioned layer, as one undo step** —
  `script.rs` `cmd_place` → `history::AddLayer`, integer `(x,y)` document-space.
- **Layer masks as a composite-time construct** —
  `LayerMask` (`layer.rs`) multiplies alpha at composite time;
  `history::AddLayerMask` attaches one. A masked, feathered paste-back is
  expressible as *placed raster layer + a `LayerMask` from the feathered
  selection*.
- **Region read/write and one-step-undo commits** — `PixelBuffer::copy_rect` /
  `paste` and `history::ReplacePixels`.
- **A serialized, sessioned, headless editor with an MCP transport** —
  `mcp/editor.rs` (single `cshop-editor` thread over mpsc) and `mcp/http.rs`
  (thread-per-connection server). The describe→draw→look→correct loop already
  carries images.

## What is missing (all on the existing roadmap)

The gaps are precisely the network/async/glue pieces the other fork docs already
sized — none is in the hard imaging core:

1. **Region extraction from a selection** into a standalone buffer. The
   primitives exist (`Selection::bounds()` + `PixelBuffer::copy_rect`); the
   single operation does not.
2. **A masked, feathered paste-back** of a rectangular buffer at `(x,y)`. Today
   `place` inserts a *maskless* opaque raster (`cmd_place`) and `ReplacePixels`
   writes wholesale with no per-pixel weighting. The composition (placed layer +
   `LayerMask`) is expressible from existing commands but is not yet a single op
   and is not scriptable.
3. **An outbound HTTP client.** The entire `mcp/http.rs` module is inbound
   server-only (a `TcpListener` accept loop; no `TcpStream::connect`, no client
   deps). This is the `cshop-net` crate the roadmap already calls for — extract
   the JSON codec and add a client.
4. **Async / background execution** so the SD round-trip does not freeze the
   window. Confirmed: **no `thread::spawn`/mpsc/async anywhere in `cshop-ui`** —
   everything runs synchronously on the egui update loop. This is the "GUI task
   runner" the roadmap flags as the gap everything else falls into.
5. **Script/command exposure** for the above: `select` is rectangle-only today,
   `export` is whole-document-only, `place` is maskless. The magic-eraser
   command would tie region-extract → downscale → inpaint-call → upscale →
   masked-composite into one undoable action.
6. **A healing/content-aware family** more broadly — today the only
   retouching-family tool is the Clone Stamp (`Tool::CloneStamp`). The magic
   eraser is a *new capability*, not an extension of it. (Classic full-res
   healing/smudge for single stray hairs remain the right non-SD tools and are
   separate work.)

The honest summary: **the pieces that are hard to build well are already
present and tested; the pieces that block this feature are the network client,
the async task layer, and the region/mask glue — and all three are already on
the roadmap for the in-app Generate work.**

## Where it lives, and the boundary implication

- **The engine is still ComfyUI, shared with the Look Editor.** The Look Editor
  sends whole ~1024 images; C-Shop sends *downscaled native-res regions*. Same
  engine, same seam, different payload sizing. C-Shop becomes a *client* of the
  same generation stack, on the technical user's procedural terms.
- **C-Shop owns the scale-down / inpaint-call / upscale / masked-composite
  wrapper.** That wrapper *is* C-Shop's answer to the high-res problem, and it is
  tractable exactly because it refuses to run SD at full resolution.
- **The natural home is the MCP/script path** (headless, sessioned, already
  threaded), with the interactive tool dispatching through the same task runner
  so the UI never blocks.

This refines the earlier boundary conclusion. It is *not* "generation is
entirely the Look Editor's and C-Shop only composites handed-in files." C-Shop
has its **own** first-class SD feature — the magic eraser — but it stays a
bounded, region-round-trip inpaint that shares ComfyUI rather than an attempt to
run diffusion over a whole 60MP frame. The boundary between the two products is
unchanged; what changes is that *both* products are ComfyUI clients, each sizing
its payload to its own resolution regime.

## Requirements this sets (the spec, condensed)

1. Region extract at native resolution from a (feathered) selection — lossless.
2. Downscale the region to a target SD size, with mask padding/context so SD has
   surrounding content to match.
3. Inpaint via ComfyUI (mask + optional prompt) at the comfortable resolution,
   over the `cshop-net` client, off the UI thread.
4. Upscale the returned region to the original region's exact pixel dimensions
   (Lanczos-3).
5. Masked composite back with feathered natural stops.
6. One undoable step at full resolution in C-Shop's history.

Only steps 3 and the downscale-before/upscale-after/masked-composite wrapping
are genuinely new; steps 1, 2, 4, 5, 6 reuse primitives that already exist and
are tested.
