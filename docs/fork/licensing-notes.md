# Licensing notes (non-determinative)

Working notes on the fork's licensing situation. **This does not decide
anything.** The fork's license will be chosen later, once it is known whether
the project is released publicly or stays a private personal tool. Until then
this file just records the facts and the options so the decision, when it comes,
is a quick one.

## Context

This fork is, first and foremost, a **personal tool** — it exists so its author
can stop using Photoshop/Lightroom and support his wife's photography workflow
on Linux. It is not (yet) a product. Public release is an open question, and the
license choice is deliberately **deferred until that question is answered**,
because the answer changes which choice makes sense.

## What upstream C-Shop grants (the fact that matters)

Upstream C-Shop is dual-licensed **`MIT OR Apache-2.0`** (declared in
`Cargo.toml` `[workspace.package] license`, with `LICENSE-MIT` and
`LICENSE-APACHE` present). The `OR` is the standard SPDX dual-license: the
project is offered under both, and **the recipient picks whichever one they
want** — satisfying either is enough. This is the usual Rust-ecosystem
convention, chosen to give downstream users maximum flexibility.

Both are **permissive**: neither is copyleft, neither forces the fork to be open
source, and both allow a proprietary or commercial product built on the code.

The only non-skippable obligation, under either, is **preserving upstream
attribution** — keep the original copyright notice ("Copyright (c) 2026 C-Shop
contributors") and the license text for the upstream code carried forward.
Practically: keep the `LICENSE-*` file(s) in the repo and don't strip the
notices. Apache-2.0 adds, on top of that, an express patent grant, a patent-
retaliation clause, and a "state significant changes" requirement (there is no
upstream `NOTICE` file to propagate).

## How the release decision maps to a license choice

When the "public or private" question is answered, this is the shortcut:

- **Stays private / personal only.** No license decision is forced at all —
  there is no distribution, so no downstream obligation is triggered. Just keep
  the upstream `LICENSE-*` files in place. Nothing else to do.
- **Released publicly, permissive.** Elect (or re-offer) `MIT OR Apache-2.0`, or
  Apache-2.0 alone. For anything commercial-leaning, **Apache-2.0 is the better
  default** because its express patent grant from the multiple "C-Shop
  contributors" is real protection MIT does not give; the cost (noting
  significant changes) is trivial.
- **Released publicly, but wanting to keep it open/copyleft.** A stronger
  license (e.g. GPL/AGPL) is possible for the fork's *own* new code, but note
  the interaction below — and that this only constrains others, not the author.

## Two things that stay true regardless of the choice

These are independent of the deferred decision and worth not forgetting:

1. **The RapidRAW / AGPL boundary is separate and still applies.** C-Shop being
   permissive means *C-Shop* imposes no copyleft. It does **not** shield the fork
   if AGPL code (RapidRAW's, or its AI-Connector's) is pulled *into* the fork —
   the AGPL would then govern the combined work. The existing "keep RapidRAW at
   arm's length: separate processes, file/HTTP handoff, no code copied" plan
   (see [`rapidraw-integration.md`](rapidraw-integration.md)) is what keeps this
   clean, whatever the fork's own license ends up being.
2. **Obligations are the union of C-Shop's terms and every dependency's.** The
   current `Cargo.toml` deps (wgpu, winit, egui, image, rayon, …) are
   overwhelmingly MIT/Apache dual-licensed, so today this is almost certainly
   clean. Before *any* public or proprietary distribution, run a license audit
   (`cargo-deny` or `cargo-about`) — and re-run it whenever a dependency is
   added — so a surprise copyleft or attribution requirement is caught before it
   ships.

## Status

**Decision: deferred.** Trigger: the public-vs-private release question. Default
lean if it goes public and commercial: Apache-2.0 (for the patent grant). No
action needed while it remains a private personal tool beyond keeping the
upstream license files in place.
