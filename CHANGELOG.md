# Changelog

All notable changes to libzstd-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-09-15

The first release: thirty-seven entry points of the libzstd C API, one
`@ffi` declaration each, and no logic.

### Added

- `libzstd` — the whole surface, in six groups.
  - The one-shot calls: `ZSTD_compress`, `ZSTD_decompress`,
    `ZSTD_compressBound`, `ZSTD_getFrameContentSize`,
    `ZSTD_findFrameCompressedSize` and `ZSTD_isSkippableFrame`.
  - The compression context: create, free, `ZSTD_compressCCtx`,
    `ZSTD_compress2`, the parameter setter, the pledged size, the
    reset, the memory figure and the dictionary loader.
  - The decompression context: create, free, `ZSTD_decompressDCtx`,
    the parameter setter, the reset, the memory figure and the
    dictionary loader.
  - The streaming interface: `ZSTD_compressStream2`,
    `ZSTD_decompressStream` and the four preferred buffer sizes.
  - The dictionary identifiers: from a dictionary and from a frame.
  - Errors and version: `ZSTD_isError`, `ZSTD_getErrorName`, the three
    compression level bounds, `ZSTD_versionNumber` and
    `ZSTD_versionString`.
- `tests/libzstd_tests.nv` — eight tests over the signatures. They call
  the C library, so they need libzstd installed. Every test compresses
  and decompresses in memory, so the suite reads and writes nothing.

### Not a `0.0.x` interface release

An interface release is the shape whose every `pub fn` body is a
`todo()`. Every `pub fn` here is an `@ffi` declaration with no body, so
`novo pkg publish` reads the package as a release with bodies and
refuses a `0.0.x` version for it. The first release of a bindings
package is therefore `0.1.0`.

### Named as missing

**Returning a structure by value.** The novo-lang foreign function
interface passes integers, floats and strings.
`ZSTD_cParam_getBounds` and `ZSTD_dParam_getBounds` return a
`ZSTD_bounds` structure by value, so a program cannot ask what a
parameter's range is; it sets the parameter and reads what
`ZSTD_CCtx_setParameter` answers instead.

**The prepared dictionary objects.** `ZSTD_createCDict`,
`ZSTD_createDDict` and the calls that take them are left out of the
first release. The copying loaders, `ZSTD_CCtx_loadDictionary` and
`ZSTD_DCtx_loadDictionary`, are here.

**The dictionary builder.** `ZDICT_trainFromBuffer` is in the separate
`libzdict` header and is a different concern from compressing.
