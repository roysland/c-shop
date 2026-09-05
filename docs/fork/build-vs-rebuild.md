# Build on C-Shop, or start fresh? A report for the AI-assisted editing workflow

Third pass, building on [`fork-assessment.md`](fork-assessment.md) and
[`modular-rewrite.md`](modular-rewrite.md). The question this time is a
product one, not an architecture one: **for a non-technical audience doing
txt2img creation + img2img/photo enhancement via Stable Diffusion, is this
codebase the right foundation, or is it cheaper to start a new, simpler
project and lift code across?**

Verified against the tree, not just the docs. The decisive technical facts
were checked in source and are cited inline.

## The short answer

**Build on C-Shop. Do not start fresh.** But the reason is narrow and worth
stating precisely, because two of your three goals lean on it and one fights
it:

- **txt2img + img2img via SD** — this project's *strongest* fit. The hard,
  boring plumbing (a sandboxed session server, place-a-PNG-as-a-layer,
  export-a-region, see-what-you-drew round trips) is **already built and
  tested over a real socket**. A new project would spend its first month
  rebuilding exactly this.
- **Basic masks + adjustments** — fully present and unusually well tested
  (GPU/CPU cross-checked). Straight win.
- **The photographer "work with a raw image" workflow** — this is where the
  codebase pushes back, and it is a *core-model* limit, not a UI one: **pixels
  are 8-bit sRGB throughout** (`cshop-core/src/pixels.rs`: "8-bit straight-alpha
  sRGB"; `PixelBuffer` is `Vec<Rgba8>`), and there is **no raw decoder** in the
  tree (no `rawloader`/`dcraw`; PSD import already "refuses 16-bit and CMYK",
  per README and the io crate). See [The raw caveat](#the-raw-caveat) — it
  changes *what* you promise, not *whether* you build here.

Starting fresh throws away ~38k lines of disciplined, tested Rust — a Vulkan
compositor, 27 cross-checked blend modes, a hand-rolled layered file format, a
PSD codec, and the entire MCP/script harness that *is* your AI integration
surface — to avoid a UI you were going to rewrite anyway. That trade is
strongly negative.

## The muscle-memory question, taken seriously

Your framing is correct and it is the reason most editors fail to convert
Photoshop users: **the shortcuts and menu geography are the product** for an
existing user, more than the feature list. Here is how the three editors sit:

| | Photoshop | Krita | GIMP | C-Shop (today) |
|---|---|---|---|---|
| Default shortcuts | — | mostly PS-compatible, painting-biased | historically divergent; PS scheme is opt-in | **PS scheme, hard-coded** |
| Menu geography | — | close-ish | notably different (Colors menu, Script-Fu) | **PS-shaped** (Image/Layer/Select/Filter) |
| Single-window UI | yes | yes | now (was infamously not) | **yes, native** |
| Non-destructive adj. layers | yes | partial | weak | **yes** |
| Onboarding for non-technical | medium | medium (painter-first) | **poor** (the "it's not Photoshop" poster child) | medium, PS-familiar |

The strategic point: **C-Shop already made the choice Krita and GIMP didn't.**
Its `shortcuts.rs` and menu bar are deliberately Photoshop-shaped (Ctrl+J
"Layer via Copy", the M/L/W/B/S/E/G/T/P tool letters, Image/Layer/Select/Filter
menus). For your stated goal — *don't lose users to "it's not Photoshop"* —
that is a large head start that a new project would have to re-derive and a
GIMP-style base would actively work against.

GIMP's lesson is the cautionary one: technically capable, free, and repeatedly
rejected by incoming users *purely* on interface unfamiliarity and a
single-window story it fixed late. Krita's lesson is the encouraging one: it
won a specific audience (digital painters) by nailing *their* muscle memory and
not trying to be everything. **Your audience is neither PS pros nor painters —
it is AI-first creators — so you inherit less muscle-memory debt than either,
and the PS-shaped defaults are "familiar enough" without being a promise you
must keep feature-for-feature.**

## Why this beats a rewrite, concretely

The instinct "copy the good bits into a simpler UI" underestimates what "the
good bits" are entangled with. What you'd be re-copying:

1. **The AI seam itself.** `cshop-app/src/mcp/` is a sandboxed, session-holding
   server with `place` (PNG → layer), `export` (region → file), selections-as-
   masks, and image-carrying results — the *exact* img2img loop, already spoken
   over HTTP and covered by socket-level tests. This is 6–10 weeks of work you
   already own. (Confirmed: `editor.rs` runs one document per session on a
   single owner thread; `tools.rs` exposes six MCP tools.)
2. **A tested pixel core.** Blend modes and adjustments implemented twice and
   compared pixel-by-pixel. You do not rebuild a compositor to get a simpler
   UI — the compositor isn't the UI.
3. **File formats.** A layered `.cshop` format and a bidirectional PSD codec,
   both round-trip- and fuzz-tested. Re-earning that trust in a new project is
   months.
4. **The modular path is already scoped.** [`modular-rewrite.md`](modular-rewrite.md)
   shows the god object and menu triplication are fixable *in place*, in
   independently-shippable phases, without touching the tested core. A rewrite
   pays the full cost of that core up front to avoid a refactor that is already
   planned and bounded.

A rewrite only wins if the existing UI is so wrong it must go entirely. It
isn't — it's PS-shaped, which is what you *want*. What's wrong is *internal
structure* (fixable) and *scope* (trimmable by removing plugin registrations,
per the modular plan). Neither justifies starting over.

## What you would trim, not rebuild

For "basic editor + AI", the fork should **subtract**, which the modular plan
makes a matter of un-registering plugins rather than deleting code:

- Drop most of the 30 filters to a handful (SD replaces most stylisation).
- Drop Pen/Shape/vector layers, advanced type, layer effects if you want
  genuinely "basic". These become disabled plugins, not dead code.
- Keep: layers, masks, selections, the core adjustments (Levels/Curves/
  Hue-Sat/WB), crop/resize, and the whole IO + MCP stack.

Then **add** the three things the assessment already sized as small:
1. A `cshop-net` crate (extract the existing MCP HTTP/JSON, add a client).
2. A GUI task runner (no `thread::spawn` in `cshop-ui` today — confirmed) so a
   20-second generation doesn't freeze the window.
3. A config store + layer provenance ("this layer came from this prompt").

A **Generate / Enhance panel** then sits on top as one command + one tool,
which — after the modular Phase 1 — is a one-file addition, not an enum
surgery.

## The raw caveat (the one real limitation)

Your photographer workflow says "working with a raw image". Be precise about
what that means here:

- **Opening a raw file (.CR2/.NEF/.ARW) directly: not supported, and not
  cheap.** No raw decoder in the tree, and the pixel core is 8-bit sRGB, so
  even after decoding you'd lose the highlight headroom that makes raw worth
  using. True raw editing implies a 16-bit/float pipeline — a *core* change
  that touches `PixelBuffer`, the compositor, and every filter's 8-bit
  round-trip. That is a large project on its own and arguably a different
  product.
- **What you can do cheaply and honestly:** accept a raw *demosaiced to 8-bit*
  (the file the camera or a one-line pre-step produces — TIFF/PNG/JPEG),
  which every "basic photo edit → SD enhance" flow tolerates fine, because
  SD img2img is itself an 8-bit round trip. For a *non-technical* audience the
  distinction is invisible: they open a photo, mask, adjust, and enhance.

**Recommendation:** promise "photo editing + AI enhancement", *not* "raw
developer". If real raw becomes a requirement later, that's a deliberate,
separately-funded core upgrade — not a reason to pick a different starting
codebase, since a new project would face the identical 16-bit core cost.

## Fit-for-audience scorecard

Your audience is **non-technical, AI-first**. Scored 1–5 for that audience
specifically:

| Need | C-Shop fit | Note |
|---|---|---|
| txt2img creation | 5 | server + place-as-layer already there |
| img2img / inpaint enhance | 5 | export-region + masks + place = the loop, built |
| Basic masks | 5 | Quick Mask, selection→mask, channels — tested |
| Basic adjustments | 5 | Levels/Curves/etc, non-destructive layers |
| Familiar UI (not-Photoshop problem) | 4 | PS-shaped already; just needs simplifying |
| Non-technical onboarding | 3 | PS-shaped ≠ beginner-simple; needs a trimmed mode |
| Raw photographer workflow | 2 | 8-bit core; "photo enhance" yes, "raw develop" no |

Six of seven are 4–5 on a foundation that already exists. That is a clear
build-here signal.

## Risks if you build here

- **The refactor must actually happen.** If phases 1–3 of the modular plan
  slip, you inherit the god object *and* pile AI features onto it. Gate new AI
  UI work behind Phase 1 (command registry) landing first.
- **Async/task layer is genuinely absent** in the UI (verified). A generation
  call on the current synchronous loop will freeze the window; build the task
  runner before any Generate panel, not alongside it.
- **`panic = "abort"`** means a bad network response can kill the app; wrap the
  client and revisit the profile before shipping.
- **UI tests need a GPU and there's no CI.** Stand up CI (even GPU-less for
  core/io/net) before growing the surface, or regressions will ride in.

## Verdict

Build on C-Shop. It is, unusually, a codebase whose *weakest* area (internal
modularity, UI scope) is the cheap-to-fix part, and whose *strongest* area (a
tested compositor, file formats, and a sandboxed AI-editing server) is exactly
the expensive part your product needs and a rewrite would have to reproduce.
Reframe the raw goal as "photo enhancement, 8-bit" to stay honest, execute the
modular plan's phases 1–2 to earn the simpler UI and the plugin seam, then add
the small net/task/config layer and land the Generate/Enhance panel as a
plugin. Starting fresh would trade a planned, bounded refactor for an unplanned,
unbounded reimplementation of the parts that are already the best in the repo.
```