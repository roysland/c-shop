---
commit: a83e6cb4558336ee9de69b0aba8ffb73d1bae72c
description: 'Codebase knowledge for module: crates/cshop-io/src (bytes, format, lib)'
files:
- crates/cshop-io/src/bytes.rs
- crates/cshop-io/src/format.rs
- crates/cshop-io/src/lib.rs
tags:
- module
timestamp: '2026-08-30'
title: crates/cshop-io/src (bytes, format, lib)
type: Module
---

### What it does

Provides binary serialization, image format detection and encoding/decoding, and document persistence for C-Shop's native and PSD formats. Handles both flat images (PNG, JPEG, etc.) and layered documents, with bounds-checking on all file reads to reject malformed or hostile files safely.

### Public interface

```rust
// Format detection and metadata
pub enum ImageFormat { Png, Jpeg, Bmp, Gif, Tiff, WebP, Tga, Ico, Cshop, Psd }
impl ImageFormat {
    pub fn from_extension(ext: &str) -> Option<ImageFormat>
    pub fn from_path(path: &Path) -> Option<ImageFormat>
    pub fn supports_alpha(self) -> bool
    pub fn is_layered(self) -> bool
    pub fn display_name(self) -> &'static str
    pub fn default_extension(self) -> &'static str
}

// Document I/O
pub fn load_document(path: &Path) -> Result<Document, IoError>
pub fn decode_document(bytes: &[u8], hint: Option<&Path>) -> Result<Document, IoError>
pub fn save_document(path: &Path, doc: &Document, composite: &PixelBuffer) -> Result<(), IoError>

// Flat image I/O
pub fn load(path: &Path) -> Result<PixelBuffer, IoError>
pub fn decode(bytes: &[u8], hint: Option<&Path>) -> Result<PixelBuffer, IoError>
pub fn encode(pixels: &PixelBuffer, format: ImageFormat, quality: u8) -> Result<Vec<u8>, IoError>
pub fn save(path: &Path, pixels: &PixelBuffer, quality: u8) -> Result<(), IoError>

// Binary serialization
pub struct Writer {
    pub fn u8(&mut self, v: u8)
    pub fn u16(&mut self, v: u16) / pub fn be_u16(&mut self, v: u16)
    pub fn u32(&mut self, v: u32) / pub fn be_u32(&mut self, v: u32)
    pub fn u64(&mut self, v: u64)
    pub fn i32(&mut self, v: i32) / pub fn be_i32(&mut self, v: i32)
    pub fn f32(&mut self, v: f32)
    pub fn bool(&mut self, v: bool)
    pub fn string(&mut self, v: &str)
    pub fn blob(&mut self, v: &[u8])
    pub fn raw(&mut self, v: &[u8])
    pub fn f32s(&mut self, v: &[f32])
    pub fn patch_be_u32(&mut self, at: usize, v: u32)
}

pub struct Reader<'a> {
    pub fn position(&self) -> usize
    pub fn remaining(&self) -> usize
    pub fn is_empty(&self) -> bool
    pub fn seek(&mut self, to: usize) -> Result<(), IoError>
    pub fn skip(&mut self, n: usize) -> Result<(), IoError>
    pub fn take(&mut self, n: usize) -> Result<&'a [u8], IoError>
    pub fn u8(&mut self) -> Result<u8, IoError>
    pub fn u16(&mut self) -> Result<u16, IoError> / pub fn be_u16(&mut self) -> Result<u16, IoError>
    pub fn u32(&mut self) -> Result<u32, IoError> / pub fn be_u32(&mut self) -> Result<u32, IoError>
    pub fn u64(&mut self) -> Result<u64, IoError>
    pub fn i32(&mut self) -> Result<i32, IoError> / pub fn be_i32(&mut self) -> Result<i32, IoError>
    pub fn f32(&mut self) -> Result<f32, IoError>
    pub fn bool(&mut self) -> Result<bool, IoError>
    pub fn string(&mut self) -> Result<String, IoError>
    pub fn blob(&mut self) -> Result<&'a [u8], IoError>
    pub fn f32s<const N: usize>(&mut self) -> Result<[f32; N], IoError>
}

pub enum IoError {
    Io(std::io::Error),
    Decode(String),
    Unsupported(String),
    TooLarge(u32, u32, u32),
    Malformed(String),
}

pub const MAX_DIMENSION: u32 = 65_536;
```

### Key invariants

- Every `Reader` operation bounds-checks before accessing bytes; invalid positions or insufficient remaining data return `IoError` rather than panicking.
- A flat image decoded from any format becomes an 8-bit straight-alpha sRGB `PixelBuffer`.
- Formats without alpha (only JPEG) are composited onto white before encoding, never dropped to black.
- Layered documents are identified first by magic bytes (`"CSHOP\0"` or `"8BPS"`), then by file extension if needed; a file retains its true format even if renamed.
- The `patch_be_u32` method assumes the caller has pre-allocated exactly four bytes at the given offset and will call it only once per patch site.
- Length-prefixed strings and blobs are serialized as a 4-byte little-endian count followed by raw bytes.

### Non-obvious decisions

- **Big-endian methods alongside little-endian in the same `Reader`/`Writer` structs**: PSD files use big-endian encoding while C-Shop's native format uses little-endian. Rather than separate reader types, both byte orders coexist in the same implementation, allowing seamless mixing when parsing hybrid formats.
- **`MAX_DIMENSION` as a pre-decode guard**: The check happens before the image decoder runs, preventing a malformed header from triggering multi-gigabyte allocations. This moves security validation to the boundary, not inside the decoder.
- **Flat images become single-layer documents**: When `decode_document` encounters a non-layered format, it wraps the pixels in a transparent-background `Document` with a `Background` layer. This unifies the API so callers always receive a `Document` regardless of input format.
- **`hint` parameter for format detection**: TGA and ICO have no reliable magic bytes, so the filename extension serves as a fallback when magic-byte sniffing fails. Magic bytes are checked first to avoid misleading extensions overriding the true format.
- **`patch_be_u32` for forward references in PSD**: PSD format requires writing a section length before the section is complete. Rather than buffering or a second pass, the code reserves space, writes the section, then patches the length in place — a common technique in binary format writers.

### Unclear intent

- The `project` and `psd` modules are referenced in `lib.rs` but their contents are not provided; their read/write signatures and the distinction between them cannot be verified against this module's public interface alone.
