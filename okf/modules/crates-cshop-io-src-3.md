---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-io/src (psd)'
files:
- crates/cshop-io/src/psd.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-io/src (psd)
type: Module
---

### What it does

Reads and writes PSD (Photoshop Document) files, supporting RGB 8-bit-per-channel images with layers, groups, blend modes, opacity, visibility, clipping, layer masks, and a flattened composite. The implementation handles PSD's non-nested layer representation (groups encoded as marker pairs) and PackBits run-length compression.

### Public interface

```rust
pub fn write(doc: &Document, composite: &PixelBuffer) -> Result<Vec<u8>, IoError>
```
Serializes a document and composite image as PSD bytes.

```rust
pub fn read(bytes: &[u8]) -> Result<Document, IoError>
```
Parses PSD bytes into a document. Falls back to treating the flattened composite as a single background layer if no layer section is present or readable.

### Key invariants

- All file data is big-endian per PSD specification.
- Layers are stored bottom-to-top in the file; a group is represented as a `GroupStart` marker (bottom divider, section type 3) followed by children, then a `GroupEnd` marker (header with name, section type 1 or 2).
- Only RGB colour mode (mode 3) at 8 bits per channel is supported; other formats are rejected.
- Canvas dimensions must be between 1 and `MAX_DIMENSION` pixels per side; version 1 PSD caps at 30,000 per side.
- Layer records with zero-sized rectangles (groups) still require four zero-sized channels in the format.
- Mask planes are stored as a fifth channel (id -2) only when present and non-empty.
- Pascal strings are padded to multiples of 4 bytes; additional-layer-information blocks are padded to even length.
- The flattened composite is always present and written even if no layers are successfully read.

### Non-obvious decisions

- **Fallback to composite on layer failure**: If the layer section is absent or unparseable, the reader reconstructs a document from the flattened composite image as a single background layer rather than failing entirely. This matches Photoshop's own robustness: a file with corrupted layer data is still viewable.
- **Negative layer count for alpha channel**: The layer record count is stored as a signed 16-bit integer; the sign indicates whether the first channel is alpha. The code uses `unsigned_abs()` to extract the magnitude regardless of sign rather than branching on it, accepting both conventions.
- **Separate handling of name and unicode name**: Both ASCII (Pascal string) and UTF-16 (luni block) layer names are read and stored. The code prefers UTF-16 if present, overwriting the ASCII name, because modern readers use the unicode version. This is stored in the file for compatibility but the unicode version is authoritative.
- **Explicit channel id mapping**: Channels are identified by fixed id numbers (-1 for alpha, 0–2 for RGB, -2 for mask) written into the record rather than inferred from position. The code iterates and matches by id rather than assuming order, making it robust to channel reordering or omission.

### Unclear intent

None identified. All major structures and transformations (layer flattening, group marker insertion, PackBits encoding, mask storage, additional-info blocks) are either standard PSD format requirements or documented in comments.
