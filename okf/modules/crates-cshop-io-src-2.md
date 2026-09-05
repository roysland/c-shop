---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-io/src (project)'
files:
- crates/cshop-io/src/project.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-io/src (project)
type: Module
---

### What it does

Serializes and deserializes C-Shop's native layered project format (`.cshop`). Documents are written as a magic header, version number, and a sequence of tagged chunks containing the layer tree, raster/vector/text/adjustment content, masks, layer effects, and alpha channels. The format is designed for forward compatibility: readers skip unrecognized chunks rather than failing.

### Public interface

```rust
pub fn write(doc: &Document) -> Vec<u8>
pub fn read(bytes: &[u8]) -> Result<Document, IoError>
```

### Key invariants

- The file format begins with the magic string `"CSHOP\0"` followed by a `u16` version number.
- Every chunk is tagged with a 4-byte identifier, followed by a 32-bit length and payload; unknown chunks are safely skipped.
- Layers are written depth-first with parents before children, allowing readers to attach children as they iterate.
- Pixel and mask data is compressed using DEFLATE (level 6); the uncompressed size is stored so readers can validate and pre-allocate buffers.
- The version number is only bumped when a change cannot be expressed as a new chunk type; format changes must remain backward compatible.
- Active layer ID and selection state are preserved so the document reopens exactly as the user left it.
- Blend modes are stored by discriminant; unknown values fall back to `Normal` rather than rejecting the file.
- Path shapes store counts before flat runs of data to prevent misalignment if a reader stops early.

### Non-obvious decisions

- Serialization is written by hand rather than derive-based, because derive macros tie the file layout to struct field order; reordering fields would silently corrupt every existing file. Explicit encoding makes format changes intentional.
- Layers are written depth-first with parents before children rather than breadth-first, because it allows a reader to insert layers into the tree incrementally without backtracking or requiring a second pass.
- Text and shape layers store their anchor/content separately from the raster fallback, and the raster is rebuilt on load from the content description. This allows the layer to remain editable even if fonts or geometry libraries differ between machines, while the saved anchor preserves visual placement when metrics diverge.
- The selection is stored as a mask buffer (pixel-aligned coverage) rather than a vector path, which simplifies serialization and aligns with the raster-based selection model.
- Path geometry is stored as counts followed by flat runs of anchors (not nested structures) so a reader that stops mid-subpath cannot misinterpret one part for another.

### Unclear intent

- The `DEFLATE_LEVEL: u8 = 6` constant is described as "the usual balance" for compression, but no benchmarks or trade-off analysis are documented. It is unclear whether this was tuned for this codebase or is a conventional default.
