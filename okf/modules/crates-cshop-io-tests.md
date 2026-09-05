---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-io/tests'
files:
- crates/cshop-io/tests/project.rs
- crates/cshop-io/tests/psd.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-io/tests
type: Module
---

### What it does

Tests the native project format (`.cshop`) for perfect round-trip fidelity and the PSD format for compatibility. Verifies that all layer properties, effects, masks, text, shapes with boolean operations, adjustments, and alpha channels survive serialization and deserialization, and that file format detection works correctly.

### Public interface

```rust
cshop_io::project::write(doc: &Document) -> Vec<u8>
cshop_io::project::read(bytes: &[u8]) -> Result<Document, Error>
cshop_io::psd::write(doc: &Document, composite: &PixelBuffer) -> Result<Vec<u8>, Error>
cshop_io::psd::read(bytes: &[u8]) -> Result<Document, Error>
cshop_io::save_document(path: &Path, doc: &Document, composite: &PixelBuffer) -> Result<(), Error>
cshop_io::load_document(path: &Path) -> Result<Document, Error>
cshop_io::decode_document(bytes: &[u8], hint_path: Option<&Path>) -> Result<Document, Error>
```

### Key invariants

- A document written and read back in the native format must be bit-for-bit identical in all layer properties, pixel data, masks, effects, text content, and shape geometry.
- Layer tree structure, order, and grouping relationships are preserved across all formats.
- The native format rejects damaged or truncated files with errors rather than panicking or allocating unboundedly.
- Unknown chunk types in the native format are silently skipped for forward compatibility with newer versions.
- File format is determined by magic bytes, not file extension; a project renamed to `.png` still opens as a project.
- PSD format carries all layer raster data and metadata but flattens to raster (groups become layer hierarchy markers, text and shapes are rasterized).
- A freshly loaded document has `modified = false`.

### Non-obvious decisions

- The native format uses length-prefixed unknown chunks rather than rejecting unrecognized data, allowing older versions to load files created by newer versions and preserve unknown chunks on re-save.
- PSD export requires passing the composite image separately rather than rendering it on-the-fly, allowing the caller to control flattening and ensuring readers that ignore layer data get the intended result.
- Format detection tries file extension before magic bytes in `save_document` and `load_document`, but magic bytes win in `decode_document`; this gives the file system hint authority when saving (respecting user intent) while ensuring data integrity when loading.

### Unclear intent

- The `rich_document()` fixture conditionally includes a text layer only if `FontDb::global().families().is_empty()` is false. This appears to be a guard against environments with no fonts, but whether this is testing recovery from missing fonts or simply avoiding test flakiness is not explicit in comments.
