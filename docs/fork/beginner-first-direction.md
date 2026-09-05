# Intuition-first companion: build on the fork, or a new project?

A separate product direction from the rest of `docs/fork/`. Those documents
treat C-Shop as a **Photoshop-familiar** editor made modular and AI-assisted.
This one asks a different question: for a specific person — **an experienced
makeup artist whose ADHD, dyslexia, and learning differences have sharpened a
powerful visual intuition** — what tool best *amplifies that intuition* while
moving the technical machinery out of her way? She is a domain expert. The goal
is not to accommodate a beginner; it is to build a professional instrument that
lets taste drive and automates the rest.

Cross-referenced with two existing projects of the user's:
- `~/Projects/ai-image/image-editor-plan.md` — the "Retouch & Inpainting Skills
  Framework", a planned Vue/TypeScript beginner-first surface.
- `~/Projects/ai-image` — the **Visual Look Editor** ("Look Studio & Recipe
  Factory"), which is **already built as a working client-side prototype** and
  is a more mature, more ambitious realisation of exactly this intuition-first
  idea (see [§7](#7-the-look-editor-already-exists)).

The short version: the new UI is not hypothetical. A strong prototype of it
exists. The question narrows to how it should relate to C-Shop and ComfyUI.

## Short answer

**This is a new project, not a fork of C-Shop's UI — and in fact you have
already built its prototype (the Look Editor). C-Shop's role is a supporting,
downstream one, not the engine.** The two are different products for different
heads:

- **C-Shop (the fork):** a real editor for someone who thinks in *layers, tools,
  and steps*. Its mental model is "here are tools; assemble a result." That is
  a *procedural* instrument — it rewards planning and sequencing.
- **The intuition-first app:** an *outcome* instrument for an expert who thinks
  in *looks* — "warmer, softer light", "deepen the lip", "make the whole thing
  feel like golden hour." She already knows what right looks like; the app's job
  is to let her express that judgement directly and handle the machinery.

The difference is not "simple vs. powerful" — it is **procedural vs.
intuitive.** A makeup artist does not compute a foundation shade; she sees it.
A procedural UI forces her to translate a seen result into a sequence of tool
operations — the translation is the tax, and it is exactly the step her
strengths let her skip. The right tool removes the translation, not the power.

You cannot get from one to the other by *hiding menus*. Reducing C-Shop's menus
and tools yields a *crippled Photoshop*, which is still a Photoshop — the
interaction grammar (select a tool → act on the canvas → tune options) is
unchanged, and that grammar is the thing that excludes her. A goal-first app
inverts the grammar: **pick an outcome → optionally point at where → see the
result.** That is a different UI, and your instinct that "this would require a
new UI" is correct.

The decisive fact: **the new UI already exists as a working prototype (the Look
Editor), and it is a web frontend, not a native egui app.** So the real choice
is not "fork vs. rewrite in Rust" — it is "wire the Look Editor you have already
built to a real generation engine, and decide what role C-Shop plays behind
it." C-Shop is one candidate for part of that engine; ComfyUI is the other, and
they are not mutually exclusive.

And there is a scope choice underneath all of this — whether generation is a
*pre-production reference* that feeds a separate, conventional human edit
(Workflow 1), or the front of a *single UI that owns every step* (Workflow 2).
The recommendation is **Workflow 1**; see [§7.3](#73-two-workflows-and-why-the-split-one-wins-first).

## Why menu-reduction is the wrong lever

A trimmed C-Shop still asks her to translate a *seen* result into a *procedure*:

1. Know that "deepen the lip" means *select the lips, make a mask, choose a
   blend mode, paint, tune opacity* — a chain of tool concepts.
2. Choose and switch tools, each with an options bar.
3. Hold the layer model in mind to know what an action even applies to.

The tax here is not the number of menu items — it is the **translation from
intuition to procedure.** That translation is precisely the step her strengths
let her skip: she perceives the finished look directly. A procedural UI makes
her stop, serialise that perception into steps, and manage state between them —
which is where dyslexia (text-dense controls), ADHD (mode-switching, held
context), and an unfamiliar abstract model (layers/masks/blend modes) each add
friction *on top of* work she does not actually need to do.

So the design principle is not "make it simpler for a beginner." It is:
**let her judgement be the input, and automate the procedure.** Remove the
translation layer, not the power.

Design implications — each one *amplifies an expert's intuition* rather than
compensating for a deficit:

- **Speak her language, not the tool's.** Not "inpaint region", but "deepen the
  lip", "soften the skin", "warm the light". A makeup artist's vocabulary *is*
  the control surface. (The Look Editor already does this — "Try the Look",
  "Nuances", "Protected Anchors".)
- **Show, don't spell.** Swatches, before/after, live preview — visual
  recognition is her fastest channel and text her slowest. This plays to
  professional taste, and it happens to sidestep dyslexia entirely.
- **Direct manipulation over parameters.** She points at *where* and nudges
  *how much* ("more of this", "less warm"); she never sets a denoise value. The
  slider is labelled in outcomes, not hyperparameters.
- **Momentum, not process.** Try, branch, compare, keep — a fast, forgiving
  loop that matches an intuitive rhythm of "try it and see", where ADHD is an
  *asset* (rapid divergent exploration) rather than a liability. Never a linear
  wizard that punishes tangents.
- **The machine keeps the books.** Files, versions, masks, seeds, undo history
  — all automatic and invisible. The technical bookkeeping that drains
  attention is the machine's job, freeing hers for judgement.

None of this is a smaller C-Shop. It is a different program that *calls* an
image engine — a professional's instrument tuned to taste, not a dumbed-down
editor.

## Three architectures, honestly compared

| | A. Trim C-Shop's egui UI | B. Look Editor + ComfyUI, C-Shop downstream | C. Look Editor + C-Shop as the primary engine |
|---|---|---|---|
| Interaction grammar | still tool-first ✗ | goal-first ✓ | goal-first ✓ |
| Reuses the built Look Editor | no ✗ | **yes — it *is* the prototype** ✓ | yes ✓ |
| Intuition-first look/feel | fights egui + PS shape ✗ | full control (web) ✓ | full control ✓ |
| Where generation lives | — | ComfyUI (locks map to its conditioners) ✓ | C-Shop MCP — but it can't do the locks ✗ |
| Effort | medium, wrong result | **low–medium, right result** | medium, forces C-Shop into a role it can't fill |
| C-Shop's role | *is* the app (wrong) | optional downstream editor ✓ | overloaded engine ✗ |

**Option B is the recommendation.** The Look Editor drives ComfyUI directly
(because the Visual Locks *are* ComfyUI conditioners), and C-Shop is a clean
*downstream* editor the result can be exported into — not the generation engine.
Option C was the tempting "one engine behind everything" idea, but C-Shop
cannot host the pose/face/light locks that make the Look Editor work, so putting
it at the centre would force it into a role ComfyUI already fills better.

## How the products relate

```
   ┌──────────────────────────┐        ┌─────────────────────────┐
   │  Look Editor (ai-image)  │        │  C-Shop (the fork)      │
   │  goal-first, verb cards, │        │  Photoshop-familiar     │
   │  Visual Locks, Recipe    │        │  layers/tools/AI editor │
   └───────────┬──────────────┘        └────────────┬────────────┘
               │ generate (locks →                   ▲
               │ conditioners)                       │ "Edit further":
               ▼                                     │ export image (+ workflow)
        ┌──────────────┐                             │
        │  ComfyUI     │─── result image ────────────┘
        │  SD / FLUX,  │
        │  ControlNet, │
        │  PuLID,      │
        │  IC-Light    │
        └──────────────┘
```

The Look Editor never shows a tool. A skill/look:

1. Picks a reference (upload/gallery), sets **Visual Locks** (keep pose/face/
   light), brushes a rough region, and picks a **swatch** — all visual, no text
   recall.
2. Its compiler turns locks + swatch + nuance sliders into a **ComfyUI job**
   (ControlNet/PuLID/IC-Light + inpaint). ComfyUI generates.
3. The result appears in the canvas; branches and "Memories" capture what
   worked; the **Recipe Card** explains how to recreate it.
4. *Optionally*, "Edit further" hands the image (and later a `run_script`
   workflow) to **C-Shop** for compositing, type, or Liquify.

C-Shop is the *adult editor downstream of a good-enough result* — not part of
generating it. That is exactly the "export a workflow to C-Shop" the user
described.

## What to reuse from C-Shop, and what not to

**Reuse (as a downstream editor, over file/MCP boundaries — not code-sharing):**
- C-Shop as an **"edit further" target**: the Look Editor exports a result and
  C-Shop opens it for compositing, type, or Liquify. File handoff works today;
  a richer `run_script` workflow handoff comes once `cshop-net` lands.
- The **software-Vulkan render path** — relevant only if the *C-Shop* side is
  later hosted server-side for the cloud/mobile ambition (main report §15). Not
  needed for the Look Editor, which renders via ComfyUI.

**Do not reuse:**
- C-Shop as the **generation engine.** Its MCP `run_script` can place/select/
  export, but it cannot host the pose/face/light **Visual Locks** — those are
  ComfyUI conditioners. Generation belongs to ComfyUI.
- The **egui UI**, `chrome.rs`, tools, menus, shortcuts — none of it fits a
  goal-first, dyslexia-friendly, touch-first surface. The Look Editor is the
  right front end and already exists.
- The **`Tool`/`Action` grammar** as a user-facing concept. It stays internal to
  C-Shop; the Look Editor speaks looks and locks, not tools.

**Borrow as a pattern, not code (licence-clean):** the *skills/category
registry* in the Look Editor is the same shape as C-Shop's filter-catalog and
the proposed command registry — one file per capability, statically listed.
Good instinct, independently arrived at on both sides; keep it in TypeScript
where it lives.

## 7. The Look Editor already exists

The `ai-image` project already contains a working prototype — the **Visual Look
Editor / Recipe Factory** — that is a stronger version of the beginner-first UI
than the generic skills framework. It is not a sketch; it runs client-side
today. Its design decisions independently match every accessibility principle
above, which is strong validation that the direction is right:

- **Goal-first, stylist-native vocabulary.** "Try the Look" not "resolve
  inpainting queue"; "Nuances" not "hyperparameters"; "Protected Anchors" not
  "conditioning constraints." Verbs and outcomes, never tools.
- **Visual recognition over text recall.** Swatch cards and preset grids, not
  adjective dropdowns — directly dyslexia-friendly.
- **Visual Locks.** One-tap toggles to *keep* pose / face / light / setting /
  feeling while changing one thing. This is the single most important idea for
  this audience: it encodes "keep what's working, change only this" without any
  layer or mask vocabulary.
- **Non-destructive branching + implicit "Memories".** Try wild variations
  without fear; presets *emerge* from what worked instead of being filled in
  upfront — a direct fit for ADHD creative rhythm.
- **Point-and-edit masking.** The same rough-brush `ImageMaskCanvas` — the only
  manual skill asked of the user.
- **The Recipe Card.** A human-readable "how to recreate this" breakdown with a
  one-click export.

Two consequences for this whole analysis:

1. **The beginner-first app is further along than the C-Shop fork.** The hard
   UX design is largely done and validated in a prototype. What it lacks is the
   *generation backend* (its current "Try the Look" is a Canvas-2D simulation,
   per its own report), which is precisely where an engine — ComfyUI, and
   optionally C-Shop — plugs in. That is its Phase 2.
2. **Its Visual Locks map onto real ComfyUI conditioners, not onto C-Shop.**
   Keep-pose → ControlNet/OpenPose; keep-face → PuLID/IP-Adapter; keep-light →
   IC-Light. These live in ComfyUI, which sharpens the "in vs. beside" question
   below: the generative intelligence is ComfyUI's; C-Shop's possible role is
   downstream document management, compositing, and export — not the locks.

### 7.1 The reference-and-recreate workflow is the genuinely novel part

The user's two-directional intent — **(a)** create an idea from txt2img,
adjusting areas/lighting; and **(b)** ingest a sample image and *extract* what
makes it work (makeup, lighting, clothing) into a "definitive reference", which
then resolves into **how to recreate it in the real world** (flash placement,
camera configuration, studio vs. outdoor) — is the part **no existing tool
does**, and it is where this product could be genuinely distinctive.

- Photoshop/Krita/GIMP: pixel editors, no notion of "recipe".
- RapidRAW: develops raw, no generative recipe extraction.
- ComfyUI/A1111: generate, but expose the machinery rather than a recreation
  plan.
- The Look Editor's **Recipe Card** already gestures at this; its roadmap notes
  **VLM recipe extraction** (LiteLLM) to read a generated look back into
  structured metadata.

Extending the Recipe Card from "creative ingredients" to a **physical
recreation plan** (key-light angle and modifier, fill ratio, background
distance, lens/aperture/ISO, indoor-flash vs. outdoor-natural) is a VLM +
templating problem layered on top of the existing card — not an image-engine
problem. It sits almost entirely in the `ai-image` project, uses its existing
LiteLLM integration, and needs *nothing* from C-Shop. This is worth calling out
because it is the feature most likely to make the product worth building at
all, and it does not depend on the fork.

### 7.2 Designing for an expert whose neurodivergence sharpens intuition

The most important reframe in this whole analysis: **she is not a beginner to
be accommodated — she is a domain expert whose ADHD/dyslexia have honed an
intuitive, visual, associative way of working.** The product should be built to
*amplify* that, treating her differences as the source of her edge rather than
as constraints to work around. Three consequences:

1. **Her makeup expertise is a design asset, not just the subject matter.** She
   already has an internal model of skin, tone, light, and how a face reads.
   The tool should *borrow* that model, not teach its own. Concretely:
   - The vocabulary of swatches and adjustments should be *her craft's*
     vocabulary — "warm the undertone", "diffuse the key light", "matte the
     T-zone" — not generic photo-editing terms. This makes the controls feel
     like extensions of her hands, and it is something a non-expert PM could
     never author. **You author the skill library with her**, in her words.
   - Because she can *judge* a result instantly, the loop should optimise for
     "generate a few, let her pick" over "specify parameters precisely up
     front." Her taste is the ranking function; the machine's job is to give
     her good options fast.

2. **Automate the technical, expose the tasteful.** Draw the line deliberately:
   - *Fully automated (she never sees it):* masking precision, seeds, denoise,
     CFG, sampler, model choice, file/version management, mask feathering,
     resolution. These are the "translation" costs.
   - *Hers to drive (front and centre):* which region, which direction ("more
     of this / less of that"), which of several results, when it's right. These
     are judgement calls only she can make.
   - The Look Editor's "Nuances" sliders and its hidden "technical generation
     settings" accordion already draw exactly this line — keep that discipline
     and resist leaking parameters upward.

3. **Fit the rhythm, don't fight it.** An intuitive, ADHD-leaning creative
   rhythm is *fast, divergent, non-linear*: try five things, keep the one that
   sings, branch off it, come back. The Look Editor's **branches + implicit
   Memories** already match this — divergent exploration with no penalty for
   tangents and no upfront form-filling. This is where her wiring is an
   advantage; the tool should reward it, never force a linear wizard.

The test for every feature: *does this let her spend more attention on taste and
less on bookkeeping?* If yes, build it; if it adds a technical decision she'd
have to hold in her head, automate it or hide it.

### 7.3 Two workflows, and why the split one wins first

There are two ways to build this, differing not in features but in *how much of
the pipeline lives in one UI*. This is the pivotal scope decision.

**Workflow 1 — generation as pre-production reference (recommended first).**
She uses the Look Editor to design the shoot *before it happens*: scene, model,
pose, makeup, lighting — converging on a single **generated reference image**
plus its **recreation recipe** (how to light and shoot it for real). Then the
real shoot happens, and the *human editing phase is separate and conventional*:
you develop the raw in Lightroom/RapidRAW, then retouch in an editor (C-Shop).

- The Look Editor's job ends at "here is the target and how to make it."
- The generated image is never the deliverable — it is a **brief**. That
  reframes everything: 8-bit output is fine, identity drift is fine, hands are
  fine, because nobody ships the generation. It only has to communicate intent.
- This is a *smaller, sharper* product than the previous framing, and it is the
  one that most directly serves her strength: she was already carrying the look
  in her head; now she can point at it. The upgrade over "image in her mind" is
  **a shared, external, referenceable artifact** — which also lets the two of
  you communicate about the shoot precisely.
- **Effort: low, and mostly already built.** It is the existing Look Editor +
  the recreation Recipe Card (§7.1). No new editing surface. The post-shoot
  path is *off-the-shelf tools you already know* (Lightroom/RapidRAW → C-Shop),
  with you as the editor. Nothing in this workflow requires C-Shop to change,
  and it does not require the ambitious single-UI integration at all.

**Workflow 2 — one UI that owns every step (much harder, later or never).**
The same design/generate stage *and* the raw develop *and* the retouch, all in
one surface she drives end to end, with control of every step.

- This collapses three mature, specialised tools (ComfyUI, a raw developer, a
  compositing editor) into one program. Each is a multi-year effort in its own
  right; the earlier documents exist precisely because *no single tool spans
  this*, which is why the recommendation elsewhere is to **compose** them
  (RapidRAW upstream, ComfyUI for generation, C-Shop downstream) rather than
  merge them.
- It also fights the §7.2 principle. "Control of every step" *is* the
  procedural burden — raw development and retouching are exactly the technical,
  bookkeeping-heavy stages the whole design tries to keep off her plate. Giving
  her every step is not obviously a gift; for this user it may be the opposite.
- **Effort: very high, and it re-introduces the raw/16-bit and compositing
  problems the other docs deliberately delegated away.** It is the "one app to
  rule them all" that the build-vs-rebuild analysis argues against.

**Recommendation: build Workflow 1 now; treat Workflow 2 as a distant maybe,
not a goal.** Workflow 1 delivers the whole *value* — she designs the shoot from
a real reference instead of memory — at a fraction of the cost, using tools that
already exist, and it keeps the clean division of labour the rest of these
documents establish. Workflow 2's extra ambition (one seamless UI) buys mostly
*control she does not want* over stages that are *your* responsibility anyway.

If Workflow 2 ever becomes desirable, the right way to approach it is *not* a
monolith but **tighter handoffs between the three surfaces** — e.g. the Look
Editor's reference + recipe travelling with the raw file into RapidRAW, and the
retouch stage in C-Shop pre-loaded with the look's regions. That gets most of
the "one flow" feel while keeping each stage in the tool built for it. Design
the handoffs to carry metadata (see §7.1's recipe JSON), and Workflow 2 becomes
an incremental tightening rather than a rewrite.

### 7.4 The delivery stage: she owns the final export

> **Superseded in part by [`delivery-model.md`](delivery-model.md).** The
> preset table below still describes the useful *export panel*, but the
> assumption that every size is a crop/resize of one pixel-identical full-res
> master no longer holds. The refined model is two taste-gated tiers: her
> working-resolution render is the final web/social deliverable (the many),
> while print/agency images (the few she loves most) get a fresh full-res
> manual pass that **overwrites** her export — the two tiers are not required
> to be identical. Read `delivery-model.md` for the governing contract; the
> panel here is the tooling that serves Tier 1 and emits Tier 2 masters.

The human editing phase of Workflow 1 ends where she does — **she is the final
arbiter of "the picture is finished."** So the last step she touches is not
retouching, it is *export/delivery*, and it deserves a first-class, dead-simple
UI. This is a concrete, bounded feature, and — importantly — **C-Shop already
has every primitive it needs**; what is missing is a one-click preset layer on
top.

**The delivery spec (from the shoot's real deliverables):**

| Preset | Output |
|---|---|
| **Agency master** | Full-size, 100% quality JPEG (no resize) — for the model agency |
| **Instagram square** | 1:1 crop, resized to IG spec |
| **Instagram portrait** | 4:5 crop, resized |
| **Portfolio 3:4** | 3:4 crop, resized |
| **Instagram landscape** | 1.91:1 crop, resized |
| **Reels / Stories** | 9:16 crop, resized |

All six also feed the **web portfolio**, so a single "Export delivery set" action
should produce the whole family at once, into a tidy named folder.

**What C-Shop already provides (verified in source):**
- **JPEG at chosen quality** — `cshop-io` `save(path, pixels, quality)` and the
  scriptable `export PATH quality=100`. The agency master is literally this with
  no resize.
- **Crop to aspect** — the Crop tool already offers aspect presets
  (`chrome.rs`/`context_menus.rs`), currently Free/1:1/4:3/3:4/16:9.
- **High-quality resize** — `ResizeImage` with Lanczos-3 (`resample.rs`), the
  right filter for downscaling to social sizes.
- **Scripting** — `export` and `resize` are already script/MCP commands, so the
  whole set can be produced by a short generated script.

**What is missing (small, well-scoped):**
1. **Three aspect presets not yet in the crop list: 4:5, 1.91:1, and 9:16.** A
   one-line addition to the aspect table (the code already handles arbitrary
   ratios via the `Some(a) => "{a:.2}:1"` branch).
2. **A "delivery presets" concept** — a named set of {aspect, target size,
   quality, filename suffix} that the export UI iterates. This is exactly the
   *catalog / skills-registry* pattern used everywhere else (filters, the Look
   Editor's categories): one entry per preset, statically listed.
3. **A crop *position* choice per aspect.** Cropping a portrait to 1.91:1
   landscape throws away most of the frame; she must decide *where* the crop
   sits (face-safe). Two honest options:
   - *Manual, and that is fine:* show the crop rectangle at the chosen aspect
     and let her nudge it — a single visual decision, squarely the kind of
     judgement call §7.2 says to leave to her.
   - *Assisted later:* face-detection to seed the crop, which she then nudges.
     Nice-to-have, not required for v1.

**How it should feel (per §7.2):** one screen, six labelled preset tiles with
live thumbnails of each crop, one "Export all" button. She picks the crop
position where it matters (the wide and tall ratios); everything else — sizes,
quality, filenames, folder — is automated. She never types a pixel dimension.

**Where it is built:** two viable homes, and the choice follows Workflow 1's
division of labour.
- *In C-Shop (recommended):* the delivery UI lives in the fork as a small
  export panel, because that is where *your* retouch stage already is and where
  the finished pixels live. It is a natural first "command registry" feature
  (main report Phase 1) and reuses the crop/resize/JPEG primitives directly.
  She does the final crop-and-export there as the last step of the session.
- *In the Look Editor:* only if the final image somehow flows back through it,
  which Workflow 1 does not require. Prefer C-Shop.

**Effort: low.** No new engine work — it is preset metadata plus a focused
export panel over primitives that already exist and are already tested
(`editing.rs` covers JPEG export; `transforms.rs` covers resize/canvas). This is
a strong, motivating early deliverable: it is visibly *hers*, it finishes the
real workflow end to end, and it exercises the C-Shop command-registry work the
main report wants anyway.

## Where this leaves the fork

Nothing here reduces the case for the C-Shop fork in the other documents, but
with Workflow 1 (§7.3) as the chosen path, the roles become unusually clean —
because the generated image is a **pre-production brief, not a deliverable**:

- **ComfyUI** is the generation engine for the *design* stage — txt2img, pose,
  makeup, lighting, every Visual Lock (ControlNet / PuLID / IC-Light). The Look
  Editor drives it.
- **The Look Editor** (`ai-image`) is her surface: design the shoot, converge on
  a reference image + recreation recipe. Its output feeds the *real* shoot, not
  a file pipeline.
- **RapidRAW / Lightroom / Darktable** develops the *real* raw photograph after
  the shoot — a separate, conventional, human stage. (You already know
  Lightroom/Photoshop; the point of the fork is a **Linux-native** alternative,
  and any of these solves the raw step.)
- **C-Shop (the fork)** is where *you* retouch the developed photograph — **and
  where she does the final crop-and-export (§7.4).** So C-Shop is not purely
  yours: it hosts the one step in the human phase that is unmistakably hers, the
  delivery. That makes the export panel the fork's most audience-relevant
  feature.

The two halves are deliberately decoupled: the AI half produces intent (a brief
she can point at); the human half produces the final image from real photography
(largely yours, with the finish-and-deliver call hers). They meet on the *set*,
not in a file format — which is why Workflow 1 needs no heroic integration.

Four surfaces, layered by how much control the user wants:

| Surface | Audience | Engine behind it |
|---|---|---|
| **Look Editor** (`ai-image`) | your wife (expert makeup artist); other intuitive creatives | ComfyUI (+ VLM for recipes) |
| **C-Shop fork** | you; prosumers; Photoshop-refugees | its own compositor + MCP AI loop |
| **RapidRAW** | photographers | its own 32-bit raw pipeline |
| **ComfyUI** | (never user-facing here) | itself |

## Recommendation

1. **Choose Workflow 1: generation as a pre-production reference (§7.3).** She
   designs the shoot in the Look Editor and gets a reference image + recreation
   recipe; the real shoot follows; the human editing phase (raw develop →
   retouch) is separate and yours. Build this. Treat the all-in-one Workflow 2
   as a distant maybe, approached later only as tighter handoffs — never a
   monolith.
2. **The intuition-first product is the Look Editor you already built** — its
   Phase 2 (wire the mask + locks to ComfyUI) is the highest-value next step and
   needs nothing from C-Shop.
3. **Co-author the skill vocabulary with her, in makeup-artist language (§7.2).**
   Her craft's vocabulary is the control surface and the product's moat; it is
   the one thing no generic tool can copy. Do this before building more skills.
4. **Put ComfyUI, not C-Shop, at the centre of generation.** The Visual Locks
   map onto ComfyUI conditioners; that is where the generative intelligence
   lives. Keep C-Shop out of the generate loop.
5. **Automate the technical, expose the tasteful (§7.2).** Hide seeds/denoise/
   masks/versions entirely; surface only region, direction, and choice-of-
   result. Optimise the loop for "generate a few, let her pick".
6. **The recreation Recipe Card is the keystone of Workflow 1 (§7.1).** A
   generated look is only useful as a brief if it says *how to shoot it for
   real*. It lives in `ai-image` with the existing LiteLLM integration.
7. **Keep the human editing phase on Linux-native tools you own.** RapidRAW/
   Darktable for develop, C-Shop for retouch — no integration required for
   Workflow 1. (The driver here is a Linux alternative to Lightroom/Photoshop,
   which you already know; the fork provides the retouch half.)
8. **Build the delivery export panel in C-Shop (§7.4) as an early, her-facing
   win.** Add the 4:5 / 1.91:1 / 9:16 crop presets and a one-click "export
   delivery set" (agency master + social crops) over the existing crop/resize/
   JPEG primitives. It finishes Workflow 1 end to end, it is visibly hers, and
   it doubles as the fork's first command-registry feature.
9. **Sequence for personal value:** Look Editor Phase 2 (ComfyUI wireup) first;
   the recreation Recipe Card second; the C-Shop delivery export panel third;
   refine the skill vocabulary with her throughout. Everything else is optional.

## Honest risks and unknowns

- **The "in vs. beside the loop" question is now answered: beside.** The Look
  Editor's Visual Locks map onto ComfyUI conditioners (ControlNet/PuLID/
  IC-Light), so ComfyUI owns generation and C-Shop is downstream. This is
  cleaner than the shared-engine idea, but it means **C-Shop delivers little to
  this specific audience** beyond the optional "edit further" handoff. That is
  fine — the fork earns its keep with the *other* audiences — but don't justify
  the fork on the beginner app's behalf.
- **The export handoff needs a real target.** "Export a workflow to C-Shop"
  works today at the file level (Look Editor writes a PNG, C-Shop opens it). A
  richer handoff (carry regions/edits as a `run_script` workflow) depends on
  C-Shop's `cshop-net`/task work landing first (main report roadmap). Ship the
  file handoff first; treat the workflow handoff as a later nicety.
- **Two/three products is real maintenance surface.** The Look Editor, the
  C-Shop fork, and (upstream) RapidRAW are separate. They share nothing but
  file/HTTP boundaries, which keeps them decoupled — but it is still three
  things. Start and prove the Look Editor's ComfyUI wireup before spreading
  effort.
- **The recreation recipe is unproven.** Turning a look into a *physical* studio
  plan (flash placement, camera settings) via VLM is the distinctive bet but
  also the least validated. Prototype it on a handful of known looks and check
  the advice against your own photography knowledge before trusting it.
- **Fit-to-her is a design discipline, not a feature.** Whether the tool truly
  amplifies her intuition can only be learned by watching her use it — her
  judgement is the spec. The Look Editor prototype is the ideal thing to sit her
  in front of next, and to co-author the skill vocabulary with. Build one real
  makeup flow end-to-end in her words, watch, and let her reactions drive the
  rest.
```