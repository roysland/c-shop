# Delivery model: her vision leads, taste gates the print pass

A follow-up to [`project-boundaries.md`](project-boundaries.md) and
[`beginner-first-direction.md`](beginner-first-direction.md) §7.4. It replaces
the earlier implicit assumption that every delivered size is a crop/resize of
one pixel-identical full-res master. That contract is dropped. In its place:

> **The edit is the truth; the pixels are a rendering of it. Her working-
> resolution export is the canonical deliverable for most images. For the few
> she loves most, the technical user redoes them to print precision on the
> full-resolution source and overwrites her export. Her taste is the gate
> between the two tiers.**

This makes her workflow the guiding one, not a pre-production brief.

## The old contract, and why we drop it

The delivery-panel sketch assumed one authoritative full-res render, cropped and
resized six ways (agency master, IG square/portrait/landscape, reels). Those
outputs had to match because they *were* the same pixels at different sizes.

That contract is what forced C-Shop to be the authoritative renderer and forced
her generation to be "just a brief." Removing it inverts the pipeline: **she
edits at her comfortable (generation) resolution and her export is the truth**,
and the high-quality print result is *derived from her intent* — not the other
way around.

## Two tiers, defined by economics not by fidelity

The insight that makes this safe: the split between tiers is an **effort-
allocation** decision, not a technical fidelity guarantee. We do not ask the
machine to reproduce her generative edits faithfully at high resolution. We let
manual effort decide *when* the divergence matters.

### Tier 1 — Web / social (the many): her render is final

- Her working-resolution export **is** the deliverable.
- No print-quality reproduction, no manual pass, zero work from the technical
  user.
- SD-resize to social sizes is fine: at web dimensions the pixel difference is
  low and nobody pixel-peeps. Generative regions stand exactly as she approved
  them.
- This is most images.

### Tier 2 — Print / agency (the few): a fresh manual edit that overwrites

- Selected **by her taste, or by a provenance requirement** — the images she
  loves most earn a print pass, and so does anything a venue needs delivered
  non-AI (see below).
- Print/agency work is *already* a fresh, precise, long edit on the full-res
  source. That is simply what print retouching is.
- So there is **no reproduction-fidelity requirement to preserve.** The gap
  between her SD-inpainted region and the print region is not a bug to close —
  it is the reason the manual pass exists. The technical user redoes that region
  by hand at full resolution regardless.
- When finished, the print master **overwrites her initial export** in place.

### Non-AI provenance: the reason Tier 2 works even under strict disclosure

Some print venues — agencies, magazines, competitions — already ask for
disclosure of AI-generated content, and some require the deliverable to be
**non-AI outright.** This constraint is tightening (Content Credentials / C2PA,
editorial standards, contest rules), so treat it as a requirement that *will*
arrive, not a hypothetical.

The two-tier model satisfies it almost by construction, and this is arguably its
strongest justification:

- Her recipe is a **description of intent — an action flow — not AI-generated
  pixels.** "Warmed the skin, removed this object, deepened the lip, lit it this
  way" is a set of instructions, not a diffusion output.
- For a non-AI print, the technical user takes **only that action flow** and
  reproduces it **manually** on the real, camera-captured full-res source —
  healing brush, clone, dodge/burn, curves, hand masking. **No SD touches the
  deliverable.**
- The provenance chain of the delivered file is therefore clean: *real
  photograph → human edit guided by a human brief → done.* Nothing to disclose,
  nothing to tag.

So her AI work becomes **pre-visualisation**: generation decides *what the
picture should be*, and that decision travels as instructions. The camera
captures reality; the technical user edits reality by hand to match her vision.
The AI never enters the delivered pixels.

**This adds no cost.** The manual print pass was already happening for quality;
the same hand-editing that yields print quality also yields non-AI provenance.
Two requirements, one pass.

**The discipline is about authorship, not pixel origin.** The line for a non-AI
deliverable is not "no pixel may trace to a model" — it is "the *picture* is a
human-authored photograph and human edit." Those are different things:

- **AI deliverable** = a model authored the image, or a substantial region of
  it, as its own output. This is what disclosure rules actually attach to:
  *generative authorship*.
- **AI-as-custom-asset** = a human uses an AI-originated image as one source
  among many in a manual composite — masking, warping, cloning, blending a patch
  of it into a real photograph. This is **just sourcing**, conceptually identical
  to using a stock texture, a frame from another exposure, or a custom brush. The
  healing brush samples from elsewhere; the clone stamp samples another source;
  masking in part of an AI render is the same manual operation with a different
  sampled source. The origin of the source pixels no more makes it an "AI edit"
  than sampling a stock photo makes it a "stock-photo edit."

So Tier 2 does **not** forbid AI-sourced assets. It forbids *generative
authorship of the deliverable*: no diffusion pass produces the image or a
wholesale region of it. A human masking a bit of a rendered plate into a camera
original is compositing, and the result is human-authored.

**The one real check is per-venue, not per-workflow:** know the venue's
definition. Normal "no AI" means no generative authorship, and AI-as-custom-asset
is fine and needs no special handling. Only the strictest few competitions define
"no AI" broadly enough to bar even AI-sourced compositing elements — that is a
rule-reading question for that venue, not a property of this pipeline.

### Object removal: the same operation as content-aware fill

The definition of "AI" is genuinely blurred at the mechanism level, and this
matters most for **object removal.** Content-aware fill, the healing brush, and
SD inpainting sit on one spectrum: all three look at surrounding pixels and
synthesise plausible pixels to fill a hole. The difference is only the *prior* —

- **healing brush**: blends a sampled source region's texture and tone,
- **content-aware fill** (PatchMatch-style): stitches in patches found elsewhere
  *in the same image* — a statistical model of the image's own content,
- **SD inpaint**: a learned prior over its whole training set.

The prior differs; the *operation* — remove object, synthesise a plausible fill
— is the same, and all three are statistical reconstruction rather than captured
pixels. The "synthesised pixels aren't real" line was crossed 15 years ago by
content-aware fill, which nobody calls AI and which is standard retouching. So
for object removal, SD-inpaint and content-aware fill really are not different
operations.

**But the scope distinction is what venues actually gate on, so keep it
explicit:**

- **Removal / cleanup is subtractive** — a stray hair, sensor dust, a
  distracting sign, a blemish. It reconstructs what the surroundings already
  imply and invents no new subject matter. On this operation SD-inpaint and
  content-aware fill are equivalent and equally accepted. **Confidence here is
  well placed.**
- **Adding subject matter that was not there** (a person, a new background, an
  object) is the *generative authorship* venues object to — and that is true
  whether done by SD or by heavy manual compositing. This is a different claim,
  and not one this workflow makes on the deliverable.

Consequence for the tiers: for the common case (removal/cleanup), the Tier 2
manual pass is largely a **quality** choice (full-res precision) rather than a
**provenance** necessity — an SD removal is defensibly equivalent to
content-aware fill. Strict-provenance rebuilds by hand are the rare case, not the
rule. (The exception is documentary/photojournalism, where *any* pixel alteration
is barred — but that excludes the healing brush too, so it is not an AI-specific
line and does not touch beauty/fashion/agency work.)

## Why this emphasises her vision more, not less

- She edits *everything* at her comfortable resolution and expresses her vision
  fully. Her export is the canonical intent and, for most images, the final
  artifact.
- **She is the branch point.** Which images get promoted to print is her
  judgment; the technical user's effort follows her taste, not a pipeline rule.
  A venue's non-AI requirement is the *other* trigger — either reason promotes an
  image to Tier 2, and the same manual pass serves both.
- His role is a **promotion step on a curated subset**, not a gate on
  everything: take her chosen export, redo it to print precision, overwrite. He
  finishes the few she elevates; he does not review the many.

Pipeline: *her vision → web deliverable (done) → [she picks favourites, or a
venue needs non-AI] → his manual print pass from her action flow → overwrite →
print deliverable.*

Seen from provenance, her AI work is **pre-visualisation**: it decides *what the
picture is*, and that decision travels as instructions the technical user
executes by hand on a real photograph. Her vision leads the whole chain even
when the delivered pixels contain no AI at all.

## What this simplifies (a whole mechanism dropped)

Earlier drafts of this idea worried about "best-effort reproduction of
generative edits at high resolution" and proposed caching each generated patch
so it could be upscaled rather than regenerated. **That mechanism is not
needed.** For web, her render stands as-is; for print, the manual pass *is* the
high-res reconstruction. So:

- **No cache-and-upscale of generated patches** for reproduction.
- **No automatic high-res replay** of generative steps.
- **No human gate on the many** — only deliberate manual work on the few.

Less to build, and it maps directly onto what C-Shop already is: its
`Action`/script/style machinery already replays *deterministic* edits (crop,
tone, colour, masks) resolution-independently, and styles already scale
themselves to any image size. The parts that would *not* reproduce faithfully
(the generative regions) are exactly the parts the manual print pass rebuilds
anyway.

## The one thing to keep deliberate: recipe travels as a brief

Keep her recipe / action list attached to (or alongside) her export — **not for
replay-reproduction, but for reference.** When the technical user sits down to
redo a favourite at print resolution, her recipe ("warmed the skin here, removed
this object, deepened the lip") is a precise brief, so his print master lands on
*her* look rather than his reinterpretation of it. That is what keeps her vision
guiding even through his hands.

So the recipe is intent-as-brief, not a reproduction contract.

## The refined seam and lifecycle

The seam stays a clean file/data handoff (no shared code), with a defined
lifecycle:

- **Payload:** her working-resolution export + her recipe/action list (as a
  brief). Optional recipe metadata as before.
- **Lifecycle of a filename:** her export is the artifact. For promoted images,
  his print master **overwrites it in place** — one filename, her-then-optionally-
  him. Un-promoted images never change.
- **Direction:** her export flows out; for the print subset, C-Shop is where his
  full-res manual edit happens, guided by her recipe, and the result replaces
  the export.

## The refined delivery contract (condensed)

- **Web/social:** her render is final; no reproduction contract, no manual work;
  SD-resize as needed. (The many.)
- **Print/agency:** a fresh full-res manual edit, guided by her recipe as a
  brief, overwriting her export when done; precision is the point, so there is
  no fidelity-to-the-draft requirement. (The few.)
- **Non-AI provenance:** because Tier 2 is a human composite authored on the
  camera original, the deliverable is human-authored — it satisfies a venue's
  non-AI requirement with nothing to disclose. The bar is *no generative
  authorship of the image*, not *no AI-sourced asset*: masking an AI-originated
  patch in as one source among many is just sourcing, like any custom asset.
  Same pass, no extra cost; only the strictest venues bar AI-sourced elements,
  which is a per-venue rule-reading question.
- **Her recipe travels with the export** as intent/brief, not as a
  replay-for-reproduction mechanism.
- **Her taste — or a non-AI requirement — is the gate** between the two tiers.

This supersedes the "one pixel-identical master, cropped six ways" assumption in
[`beginner-first-direction.md`](beginner-first-direction.md) §7.4. The delivery
*export panel* described there is still useful for Tier 1 (produce her social
crops from her render) and for emitting Tier 2 print masters, but the two tiers
are no longer required to be identical — and the technical user's overwrite, not
a crop of a shared master, is what produces the print deliverable.
