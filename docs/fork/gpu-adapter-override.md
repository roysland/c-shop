# Selecting the right GPU on multi-adapter systems

A design note, not a task. Nothing here needs building until the problem
actually shows up — the current behaviour is fine on the machines tested so
far. This exists so that when someone opens the app and it renders on the
wrong device, the fix is already thought through.

Verified against the tree. The relevant code is all in
[`crates/cshop-gpu/src/context.rs`](../../crates/cshop-gpu/src/context.rs) and
[`crates/cshop-app/src/window.rs`](../../crates/cshop-app/src/window.rs).

## What happens today

C-Shop never enumerates GPUs or lets the user choose one. It asks `wgpu` for a
single adapter and takes whatever comes back. From `context.rs`:

```rust
.request_adapter(&wgpu::RequestAdapterOptions {
    power_preference: wgpu::PowerPreference::HighPerformance,
    force_fallback_adapter: false,
    compatible_surface: surface,
    ..Default::default()
})
```

Two things are doing the work here:

- `power_preference: HighPerformance` — a *hint* that the backend should
  prefer the fastest (typically discrete) GPU.
- `force_fallback_adapter: false` — do not deliberately pick a software
  rasterizer (`llvmpipe` on Mesa).

The chosen adapter is logged, which is the only visibility into the decision:

```rust
let info = adapter.get_info();
log::info!("GPU: {} ({:?}, {:?})", info.name, info.device_type, info.backend);
```

The backend is left to `wgpu`'s environment defaults — `window.rs` builds the
instance with `new_with_display_handle_from_env(...)` and the headless path in
`context.rs` uses `new_without_display_handle_from_env()`. Neither pins a
backend or filters by vendor. The only place the code branches on the device
at all is `texture_budget()`, which keys off `device_type`
(`DiscreteGpu` / `IntegratedGpu` / other), never off vendor or name.

## What the override can and cannot do on dual-GPU systems

Plenty of people run two GPUs, and they split into two camps that want very
different things. The override serves one of them and deliberately not the
other.

The key constraint: the override pins **the entire C-Shop process to one
adapter**. Everything that touches the GPU flows through a single
`GpuContext` — one `wgpu::Device` and one `wgpu::Queue`, created once in
`context.rs` and `Arc`-cloned to the compositor, the layer texture cache, and
readback. There is no second device anywhere in the tree. So the override
answers exactly one question: which GPU that single context binds to.

- **Separate domains — supported, and the main point of the feature.** A user
  who dedicates one GPU to their display / compositor / LLM inference and wants
  C-Shop to live entirely on the other card gets exactly that: set
  `CSHOP_ADAPTER` and the whole editing workflow — compositing, uploads,
  readback, present — runs on the chosen card, with nothing leaking onto the
  other. The one seam is **present**: on the windowed path (`window.rs`) the
  chosen adapter must still present to the window's surface, and the window
  lives on whichever GPU drives the monitor. Pinning C-Shop to a card that is
  not connected to the display means `wgpu` must copy the final frame across to
  the presenting GPU, or may not be able to present at all — hence the
  surface-compatibility fallback in the proposal below. The headless / CLI
  export path has no surface and no such constraint, so pinning there is clean.

- **Shared capacity / pooling — explicitly a non-goal.** Treating two GPUs as
  one pooled resource (splitting a single composite across both cards, or
  load-balancing documents between them) is **not** something this override can
  provide, and not something the codebase can do at all today. It is an
  architecture limit, not a config gap: C-Shop is built around a single
  `GpuContext`, there is no multi-device abstraction, and cross-device textures
  are unsupported by design — the `context.rs` comment notes egui and the
  compositor deliberately share *one* device for exactly this reason. The
  compositor's ping-pong scratch buffers assume one device's memory. Multi-GPU
  pooling would be a real engineering project, not an environment variable, and
  it would buy little here: an interactive editor is latency- and fill-rate-
  bound on a single frame, where a second GPU mostly adds cross-device copy
  overhead. The usual reason to reach for a second card — running out of VRAM —
  is already mitigated by the constant-memory tiling in `compositor.rs`.

Anyone reading `CSHOP_ADAPTER` as a pooling knob is mistaken; it is a
which-one-GPU selector, nothing more.

## Why this is usually enough

`request_adapter` is not the same as a bare device list. It applies a
preference and excludes the fallback rasterizer, so on the common cases it
lands correctly without help:

- **Discrete AMD/NVIDIA + a software `llvmpipe` device.** `HighPerformance`
  plus `force_fallback_adapter: false` picks the real GPU and skips the
  software one.
- **A machine where only the integrated GPU has a usable driver** (e.g. a
  laptop with an NVIDIA card whose proprietary driver is not installed).
  `wgpu` enumerates only what is actually usable; the preference cannot conjure
  a GPU with no driver, so it uses the integrated one. This is correct, not a
  fallback failure.

This is meaningfully different from the ROCm/HIP compute stack, where the app
gets an index list and picks wrong. `wgpu`'s Vulkan path applies a heuristic
first. Same hardware, different enumeration model.

## When it will break

`power_preference` is a hint the backend is free to interpret. It can guess
wrong when:

- Multiple real adapters are present and the "fastest" is ambiguous or
  misreported.
- A phantom or duplicate device appears (a card exposed under two APIs, a
  software device that reports as something other than `Cpu`, a headless
  render node vs. the display GPU).
- The user simply wants a *different* GPU than the fastest one — for power,
  thermals, or because the fast one is busy with another workload.

When that happens there is no escape hatch. There is no env var, no `--gpu`
flag, no way to list adapters and choose. The app takes `request_adapter`'s
answer and configures the surface around it.

## Proposed fix (deferred)

Add an opt-in override, off by default, that only changes behaviour when set.
The `PowerPreference` path stays the default so nothing regresses.

Shape of it:

- Read an environment variable — proposed name **`CSHOP_ADAPTER`** — at
  context creation. Support both:
  - an **index** (`CSHOP_ADAPTER=1`) into the enumerated adapter list, and
  - a **name substring, case-insensitive** (`CSHOP_ADAPTER=radeon`), matched
    against `adapter.get_info().name`.
- When set, enumerate all adapters for the active backend
  (`instance.enumerate_adapters(...)`), pick the first match, and use it
  instead of `request_adapter`. When it names something that does not exist,
  log a warning and fall back to the current `request_adapter` behaviour rather
  than failing to start.
- Keep a companion **`CSHOP_LIST_ADAPTERS`** (or a `probe` example flag) that
  prints every adapter's index, name, `device_type`, and `backend`, then exits.
  Without a listing the user cannot know what to put in `CSHOP_ADAPTER`.
- Preserve the surface-compatibility check. On the windowed path the chosen
  adapter must still be able to present to the surface; if a hand-picked
  adapter cannot, warn and fall back.

An `.env` file is a reasonable delivery mechanism for this, but note the app
does not currently load one — nothing in the tree reads `.env`. If we want a
file rather than a shell variable, that pulls in a loader (e.g. an
env-file crate read once at startup, before `GpuContext::new`). The variable
itself is the core of the feature; `.env` support is a convenience layer on
top and can come with it or later.

### Why this mirrors what the user already knows

This gives the same escape hatch that `ROCR_VISIBLE_DEVICES` / `HIP_VISIBLE_DEVICES`
give on the compute side: an explicit, per-run override of automatic device
selection. Someone who has fought ROCm picking the wrong GPU id will reach for
exactly this, so the name and semantics should feel familiar (name or index,
set in the environment, logged on startup).

## Scope and cost

Small and localised. The change lives almost entirely in `context.rs`
(`GpuContext::new` and `headless`), plus a few lines in `window.rs` if the
surface-compatibility fallback needs wiring, plus optional `.env` loading.
No shader, compositor, or format changes. The existing `log::info!("GPU: ...")`
line already gives the verification step for free.

## How to confirm the current pick before doing anything

Run with logging and read the `GPU:` line:

```
RUST_LOG=info cargo run -p cshop-app
```

or the probe example:

```
RUST_LOG=info cargo run -p cshop-gpu --example probe
```

If it reports the intended discrete GPU with backend `Vulkan` (on Linux),
there is nothing to fix and this note stays deferred. If it reports `llvmpipe`,
a `Cpu`/`Other` device, or the wrong card, that is the trigger to implement the
override above.
