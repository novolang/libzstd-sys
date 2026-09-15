# libzstd-sys

Zstandard is a lossless compression algorithm and file format. It aims
at the compression ratio of zlib at several times the speed, and it
scales from a level that is faster than copying to a level that
compresses harder than xz. The format is specified in
[RFC 8878](https://www.rfc-editor.org/rfc/rfc8878.html) and the
reference implementation is the C library libzstd, documented in
[zstd.h](https://raw.githack.com/facebook/zstd/release/doc/zstd_manual.html).
This package declares thirty-seven of that library's entry points to
novo-lang, one declaration each.

**Status: a binding, not a port.** Every function in this package is a
declaration of a function in libzstd. The package contains no logic of
its own, and it does nothing without the C library installed. The
thirty-seven entry points are the ones a program needs to compress and
decompress, in one call or a piece at a time; the section "What is not
included" says what a program still cannot do with them alone.

## What it is

A **frame** is the unit Zstandard produces. It begins with a magic
number and a header, holds one or more blocks, and optionally ends with
a checksum of the original content. A file may hold several frames one
after another.

The **compression level** trades ratio for speed. `ZSTD_minCLevel`
answers the smallest the library accepts, which is negative and very
fast, and `ZSTD_maxCLevel` answers the largest. Level 3 is the default.
Level is the only setting most callers change.

A **context** is the working memory a compression or decompression
uses, together with the parameters set on it. Creating one costs an
allocation, and a program that compresses many things creates one
context and reuses it.

A **dictionary** is a block of sample data both sides share. Small
inputs compress badly because the compressor has no history to refer
back to; a dictionary supplies that history before the input begins,
and it is what makes Zstandard good at compressing many small records.

The **streaming interface** is for data that does not fit in memory at
once. The caller describes an input buffer and an output buffer,
libzstd consumes and produces what it can, and the caller refills and
drains. A buffer description is three values: the address, the size and
the position reached so far.

## Install

```
novo pkg add libzstd-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the library and its header come from the system package `libzstd-dev`:

```
sudo apt install libzstd-dev
```

On macOS the Homebrew formula is `zstd`. On other systems the library
builds from the Zstandard source with `make`.

## Example

A buffer compressed and decompressed again:

```novo ignore
use libzstd

fn main() [io, ffi]
    let message = "the quick brown fox jumps over the lazy dog"
    let src = ptr.alloc(64)
    ptr.write_bytes_buf(src, bytes.from_str(message))

    // A buffer of the bound never overflows, whatever the input.
    let bound = libzstd.zstd_compress_bound(43)
    let frame = ptr.alloc(bound)
    let n = libzstd.zstd_compress(frame, bound, src, 43, 3)
    if libzstd.zstd_is_error(n) == 1
        println(ptr.read_str(libzstd.zstd_get_error_name(n)))
        return
    println("${n} byte frame from 43 bytes")

    // The frame records its own decompressed size.
    let size = libzstd.zstd_get_frame_content_size(frame, n)
    let back = ptr.alloc(size)
    let got = libzstd.zstd_decompress(back, size, frame, n)
    println(bytes.to_str(ptr.read_bytes_n(back, got)))

    ptr.free(back)
    ptr.free(frame)
    ptr.free(src)
```

The example is fenced as an illustration rather than a compiled block
because `novo doc` compiles the blocks in documentation comments and not
the ones in this file. The same calls are in
`tests/libzstd_tests.nv`, where the round trip is asserted.

## What the package contains

| Module | Contents |
| --- | --- |
| `libzstd` | Every entry point, in six groups: the one-shot calls, the compression context, the decompression context, the streaming interface, the dictionary identifiers, and errors and version. |

The six groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| One-shot | 6 | Compresses and decompresses a whole buffer, and reads a frame's sizes. |
| Compression context | 8 | Creates reusable working memory, sets the parameters, and compresses with them. |
| Decompression context | 7 | The same for decompression, with the window limit that bounds an untrusted frame. |
| Streaming | 6 | Compresses and decompresses a piece at a time, and reports the preferred buffer sizes. |
| Dictionary identifiers | 2 | Reads the identifier out of a dictionary and out of a frame. |
| Errors and version | 8 | Tests and names an error, reports the level bounds and the version. |

## How to choose an entry point

`ZSTD_compress` and `ZSTD_decompress` are for a buffer already in
memory, compressed once. They allocate a context internally and free it
again.

`ZSTD_compressCCtx` and `ZSTD_decompressDCtx` are the same calls with a
context the caller keeps. Use them for the second and every later
buffer.

`ZSTD_compress2` is the one-shot call that reads its settings from the
context rather than from a level argument. Use it when anything besides
the level matters: the checksum, the window size, the worker count.

`ZSTD_compressStream2` and `ZSTD_decompressStream` are for data that
arrives or leaves in pieces. They are also the only way to produce a
frame whose size is not known in advance.

## The rules a user needs

1. **A pointer is an `Int`, and zero is null.** Every handle the C
   library returns arrives as the address it returned.
2. **A buffer is an address and a length.** The caller reserves the
   bytes with `ptr.alloc`, writes them with `ptr.write_bytes_buf`, and
   reads them back with `ptr.read_bytes_n`. `ptr.alloc` does not clear
   the bytes, so read back exactly the length the call reported.
3. **An error is a value, not a flag.** A size and an error are the
   same C type, and an error is a code subtracted from zero. Test every
   returned size with `ZSTD_isError`; comparing it with zero is not the
   same test.
4. **An entry point that answers a C `int` answers it in 32 bits.**
   Write `as i32` before comparing the answer with a negative number.
   `ZSTD_minCLevel` is the one that needs it.
5. **`ZSTD_getFrameContentSize` answers -1 and -2, and neither is a
   size.** -1 is `ZSTD_CONTENTSIZE_UNKNOWN`, which a frame written
   without a pledged size gives; -2 is `ZSTD_CONTENTSIZE_ERROR`.
   RFC 8878 section 3.1.1.
6. **A decompressed size from a frame header is a claim, not a fact.**
   The frame is data, and the header can say anything. Bound the
   allocation yourself before trusting it, and bound the window with
   `ZSTD_DCtx_setParameter` parameter 100.
7. **`ZSTD_CCtx_setParameter` answers the value it set.** For some
   parameters that is the requested value clamped into its bounds, so a
   caller that wants to know what it got reads the answer.
   `ZSTD_DCtx_setParameter` answers 0 instead.
8. **A pledged source size must be exact.** `ZSTD_CCtx_setPledgedSrcSize`
   writes the number into the frame header, and the compression fails
   if fewer or more bytes arrive. Pass -1 for unknown.
9. **A streaming buffer description is three eight-byte words**, in
   this order.

   | Offset | Meaning |
   | --- | --- |
   | 0 | the address of the bytes |
   | 8 | the size of the buffer |
   | 16 | the position reached so far |

   libzstd updates the position. The caller reads it back to find how
   much was consumed or produced.
10. **A zero from `ZSTD_decompressStream` means a frame just ended.**
    An empty input does not: a frame can be complete with bytes still
    to come, and more bytes can arrive with the frame still open. Stop
    on the zero.
11. **A decompressor needs the dictionary the compressor used.** A
    frame compressed with a dictionary cannot be read without it, and
    the failure is an error rather than wrong output.
12. **The parameter numbers are numbers**, because the C header spells
    them as an enumeration.

    | Parameter | Number | What it sets |
    | --- | --- | --- |
    | `ZSTD_c_compressionLevel` | 100 | the level |
    | `ZSTD_c_windowLog` | 101 | the back-reference distance, as a power of two |
    | `ZSTD_c_checksumFlag` | 201 | whether a checksum is written at the end of the frame |
    | `ZSTD_c_nbWorkers` | 400 | how many threads compress in parallel |
    | `ZSTD_d_windowLogMax` | 100 | the largest window a frame may ask for |

13. **The reset directives are 1, 2 and 3.** 1 ends the frame in
    progress, 2 restores the parameters, and 3 does both. There is no
    directive 0.

## What is not included

- **`ZSTD_cParam_getBounds` and `ZSTD_dParam_getBounds`.** They return
  a `ZSTD_bounds` structure by value, and the novo-lang foreign
  function interface passes integers, floats and strings.
  `ZSTD_CCtx_setParameter` answering the value it set is how a caller
  learns the same thing.
- **The prepared dictionary objects.** `ZSTD_createCDict`,
  `ZSTD_createDDict`, `ZSTD_compress_usingCDict` and
  `ZSTD_decompress_usingDDict` digest a dictionary once and reuse it.
  They are left out of the first release; the copying loaders are here.
- **The dictionary builder.** `ZDICT_trainFromBuffer` is in a separate
  header and builds a dictionary from samples. Training a dictionary is
  a different job from compressing with one.
- **The superseded streaming interface.** `ZSTD_initCStream`,
  `ZSTD_compressStream`, `ZSTD_flushStream` and `ZSTD_endStream` are
  replaced by `ZSTD_compressStream2` with its end directive.
- **The custom allocator.** `ZSTD_createCCtx_advanced` takes a
  `ZSTD_customMem` structure by value.
- **The frame header inspection interface.** `ZSTD_getFrameHeader`
  fills a `ZSTD_frameHeader` structure the caller lays out, and its
  fields have no stable offsets a binding can promise.
- **The experimental interface.** Everything guarded by
  `ZSTD_STATIC_LINKING_ONLY` is absent, because it has no stable
  application binary interface.

## Related packages

`zstd-nv` is the Zstandard codec written in novo-lang, with no C
library. This package is the reference implementation that port
measures itself against. `zstd-nv` is planned and not published yet.

Choose `zstd-nv` when the program must build for a microcontroller or
for WebAssembly, or when a C toolchain is not wanted. Choose this
package when the program needs the reference implementation's speed, or
must match its output byte for byte.

## Tests

`tests/libzstd_tests.nv` holds eight tests written against the
signatures. They call the C library, so `novo test` needs libzstd
installed and linkable:

```
novo test tests/libzstd_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

Every test compresses and decompresses in memory, so the suite reads
and writes nothing. The one-shot round trip asserts that the message
comes back byte for byte. The frame tests assert that the frame records
its own content size and its own compressed size, and that bytes which
are not a frame answer `ZSTD_CONTENTSIZE_ERROR`. The context tests set
the level, the checksum and the pledged size and assert that the two
frames differ. The streaming test lays out the two buffer descriptions
by hand and round-trips the same message through
`ZSTD_compressStream2` and `ZSTD_decompressStream`. The dictionary test
loads one raw content dictionary into both contexts and round-trips
through it.

## Implementation status

| Group | State |
| --- | --- |
| One-shot | Complete. |
| Compression context | Complete for the copying dictionary loader. |
| Decompression context | Complete for the copying dictionary loader. |
| Streaming | Complete for the current interface. |
| Dictionary identifiers | Complete for a dictionary and a frame. |
| Errors and version | Complete. |
| Parameter bounds | Absent. Both entry points return a structure by value. |
| Prepared dictionaries | Absent. Left out of the first release. |
| Dictionary training | Absent. It is a separate header. |
| Experimental interface | Absent. It has no stable binary interface. |

## Licence

Apache-2.0. See [LICENSE](LICENSE).

Zstandard itself is distributed under the BSD licence and the GPL
version 2, and installing it is the reader's own step.
