# xxhash-nv

xxHash for novo-lang: XXH32, XXH64 and XXH3, as **three separate
algorithms with three separate entry points** — because a file format
names one of them, and the wrong one is four plausible bytes that fail
on somebody else's machine.

**Status: NOT IMPLEMENTED — interface only.**

Every `pub fn` body is a `todo()`.  The signatures, the effect rows and
the types are published so the design can be reviewed and depended on
before anyone writes a round function; the first implementation is the
`0.1.0` published over this.

## What this is

xxHash is a **non-cryptographic** hash chosen for throughput.  It
detects accidental corruption very well — a failing disk, a truncated
download, a bit flipped in transit — and an adversary who can choose
the bytes can produce a collision in seconds.  Every format that uses
it says exactly that much, and so does this page.

It is **not a CRC**, either.  A CRC is a different family with a
different purpose: it is designed to catch burst errors on a wire and
has provable detection properties for them.  crc-nv is the package for
that; this one is for the formats that specify xxHash.

This package publishes:

| module | what it is | in the device claim |
| --- | --- | --- |
| `xxh32` | XXH32's state and round, in word algebra | yes |
| `xxh64` | XXH64's state and round, in word algebra | yes |
| `xxh3` | XXH3's constants, length paths and 128-bit digest | no |
| `xxhbytes` | the bridge: `Bytes` in, digests and text out | no |
| `xxherr` | one short error type, all of it XXH3's | no |

**It exists because two published packages each carry a private copy.**
zstd-nv 0.0.1 ships `zstdxxh` (XXH64) and lz4-nv 0.0.1 ships `lz4xxh`
(XXH32), and both modules open with the same paragraph: *"Searched the
registry before writing it: there is no xxhash package."*  Both ask for
this row by name.

## Install

```
novo pkg add xxhash-nv
```

## The one example that will work

```novo
use std.bytes
use xxh32
use xxh64
use xxhbytes

fn main() [io]
    let data = bytes.from_str("the quick brown fox")

    // One shot, at the width your format names.
    println(xxhbytes.text32(xxhbytes.hash32(data, 0)))
    println(xxhbytes.text64(xxhbytes.hash64(data, 0)))

    // Streaming: the state is a value and `feed` answers the next one.
    var h = xxh64.start(0)
    h = xxhbytes.feed64(h, bytes.from_str("the quick "))
    h = xxhbytes.feed64(h, bytes.from_str("brown fox"))
    println(xxhbytes.text64(xxh64.finish(h)))    // the same number
```

Build and test it with:

```
novo pkg build
novo test
```

`novo test` is red today, on purpose: every assertion reaches
`not implemented: xxhash-nv.<module>.<fn>`.

## The load-bearing interface: three algorithms, three entry points

There is no `hash(data, width)` in this package and there will not be
one.  `hash32`, `hash64`, `hash3_64` and `hash3_128` are four names a
caller types on purpose, and that is the design.

The reason is in this package's own two consumers.  **A Zstandard
frame's `Content_Checksum` is four bytes and they are the low half of
an XXH64.  An LZ4 frame's content checksum is four bytes and it is an
XXH32.**  Two four-byte checksums, over the same content, in two
formats a program may well read in the same afternoon — and they are
different numbers.  A width parameter makes computing the wrong one a
typo; four names make it a decision.

The same argument scales up.  XXH3 is not XXH64 with a better round
function: it is a different algorithm with a 192-byte secret, eight
accumulator lanes designed for SIMD, and **four separate code paths
chosen by input length**.  A caller who picked `width = 64` and got
XXH3 would produce a digest no zstd reader will ever accept.

What the three genuinely share is the family's promises and its
failure mode, and that belongs in one package so a reader meets them
once.

Two smaller decisions follow from the same place:

- **`finish` is a pure read.**  It does not consume the state, so a
  caller can report a running hash and keep feeding — which is exactly
  what a compressor emitting a per-block checksum *and* a
  whole-content checksum has to do, and what a consume-on-finish API
  would force it to duplicate.
- **`xxh64.low32` and `xxh32.byte_of` are published.**  The truncation
  a Zstandard trailer stores and the byte an LZ4 header stores are one
  line each, and both are the kind of line every consumer
  independently rediscovers and half of them get wrong.

## The three algorithms, and how they differ

XXH64 is **not** XXH32 at double the width.  A port that widened one
into the other passes nothing:

| | XXH32 | XXH64 |
| --- | --- | --- |
| stripe | 16 bytes, 4 lanes of 4 | 32 bytes, 4 lanes of 8 |
| round | accumulate, rotate 13, multiply | accumulate, rotate 31, multiply |
| merge | XOR the four rotated lanes | applies the round **again** to each lane, then `* P1 + P4` |
| tail | 4 bytes, then 1 | 8 bytes, then 4, then 1 |
| avalanche | shifts 15, 13, 16 | shifts 33, 29, 32 |
| primes | five 32-bit numbers | five different 64-bit numbers |

XXH3 shares none of the above.  Its four length paths are published as
constants — at or below 16, 17 to 128, 129 to 240, and above — because
they are the first thing an implementer lines up against the reference
and because a benchmark that covers three of four is a benchmark that
reports the wrong thing.

**XXH3 has no specification document.**  xxHash publishes an
`xxhash_spec.md` covering XXH32 and XXH64; XXH3 is defined by
`Cyan4973/xxHash`'s own source, with the guarantee that the output has
been stable since v0.8.0.  So for XXH3 the reference implementation is
not a second opinion — it is the definition, and this package's job is
to reproduce it rather than to interpret it.  Worth knowing before
somebody spends a day looking for the paragraph that settles an
ambiguity.

## The two halves, and which one a device gets

`xxh32` and `xxh64` take **no `Bytes`, no `Str` and no list at all**,
and what they give a device is **two whole hashes**: `start`,
`feed_byte`, `finish`.  A state is four accumulators and a four-word
stripe buffer inside a `@value` — 56 bytes of the caller's frame for
XXH32, 88 for XXH64 — and `tests/embedded_probe.nv` builds both for a
Cortex-M4.

The device is real and is named in the plan.  lz4-nv's row says its
block decoder is what a bootloader runs; the LZ4 frame format checks
three XXH32s (a header byte, a per-block checksum, a content checksum),
and **a bootloader that can decompress and cannot check the checksum
has verified nothing**.  zstd-nv is the same argument at XXH64.

`xxh3` is deliberately **outside** the claim.  Its accumulator layout
exists to be eight lanes of SIMD; on a Cortex-M4 the scalar path is
slower than XXH32 and costs some 256 bytes more of state and secret.  A
device that wants a checksum wants `xxh32`.

`xxhbytes` speaks `Bytes` and `Str` and is the host's half.

## Signed integers, and printing a digest

`Int` is 64-bit and signed.  Three of XXH64's five primes have their
high bit set, so they are written in the source as negative decimals
with the hexadecimal in a comment — and **a finished XXH64 is very
often a negative number**.  Printing one with `${h}` gives a minus sign
and a value no other tool has ever produced.

`xxhbytes.text32`, `.text64` and `.text128` are what `xxhsum` prints.
`text128` puts the **high** half first, which is the opposite of the
canonical byte order `digest128_into` writes — that disagreement is the
one every 128-bit hash's tooling has, and both are published so nobody
has to guess which they are looking at.

XXH32's words are 32 bits inside a 64-bit `Int`, so every product and
rotation comes back through `xxh32.mask`.  A missing mask is the defect
that produces plausible values and diverges later; `mask` is public and
the tests assert it at the round function.

## What zstd-nv and lz4-nv delete

Both packages carry a private module today.  Every function in each has
a call here, so both delete a module at their next version:

| zstd-nv `zstdxxh` | here |
| --- | --- |
| `ZstXxh64` | `xxh64.Xxh64State` |
| `start(seed)` | `xxh64.start(seed)` |
| `feed(h, chunk)` | `xxhbytes.feed64(h, chunk)` |
| `finish(h)` | `xxh64.finish(h)` |
| `hash(data)` | `xxhbytes.hash64(data, 0)` |
| `truncate32(h)` | `xxh64.low32(h)` |
| `as_unsigned_text(h)` | `xxhbytes.text64(h)` |
| `stripe_bytes()` | `xxh64.XXH64_STRIPE_BYTES` |

| lz4-nv `lz4xxh` | here |
| --- | --- |
| `Lz4Xxh32` | `xxh32.Xxh32State` |
| `start(seed)` | `xxh32.start(seed)` |
| `feed(h, chunk)` | `xxhbytes.feed32(h, chunk)` |
| `finish(h)` | `xxh32.finish(h)` |
| `hash(data)` | `xxhbytes.hash32(data, 0)` |
| `header_byte(h)` | `xxh32.byte_of(h, 1)` |
| `as_unsigned_text(h)` | `xxhbytes.text32(h)` |
| `stripe_bytes()` | `xxh32.XXH32_STRIPE_BYTES` |

Both gain something in the move: the range forms (`feed32_range`,
`hash64_range`) hash a window of a scratch buffer without slicing it,
which is the difference between one allocation and one per block in a
frame reader.

## The layer, and why

`core`, and the budget is `[]` throughout.

xxHash is multiplication, rotation and XOR over fixed primes.  The
seed, the secret and the bytes all arrive as arguments; there is no
entropy source, no clock and nothing that opens a file.  **No
dependencies**, and nothing to take one for.

## What is outside, and named

- **XXH3's streaming state.**  `Xxh3State` declares the accumulator
  bank so the shape is public, and the streaming form is not here: it
  also needs a 256-byte input buffer and a derived secret, and its
  block-accumulation rule is a second algorithm's worth of detail.  The
  one-shots cover every consumer the grid has.  It is the first thing
  to add when one asks.
- **XXH128's dispersion helpers** (`XXH128_cmp`, `canonicalFromHash`).
  `Xxh128` is two integers and comparing two integers needs no helper;
  the canonical bytes are `xxhbytes.digest128_into`.
- **SIMD.**  The reference gets its speed from AVX2 and NEON paths.
  This is a scalar port, it will be slower, and saying so is more
  useful than implying otherwise.  Nothing in the surface would change
  if a vectorised backend arrived.
- **XXH1 and XXH0.**  Superseded by the algorithm's own author.
- **A CRC.**  Different family, different promise: crc-nv.

## The reference implementation

`Cyan4973/xxHash` is the oracle.  For XXH32 and XXH64 its
`xxhash_spec.md` is the written definition and the source is the second
opinion; for XXH3 the source is the definition, as above.

The tests carry:

- **The empty-input digest at all four widths** — `02cc5d05`,
  `ef46db3751d8e999`, `2d06800538d394c2` and
  `99aa06d3014798d86001c324468d497f`.  The value every port prints
  first, and the one every implementation agrees on before it agrees on
  anything else.
- Every structural property that does not need a table: chunking
  invisibility across a stripe boundary, seed sensitivity, the four
  XXH3 length paths each exercised, the canonical byte orders, and the
  two consumers' exact calls.

**The reference's full sanity table is not transcribed here.**
`xsum_sanity_check` hashes prefixes of a 2367-byte buffer produced by
its own PRNG, at several seeds, and the table runs to over a hundred
values.  Copying that many numbers by hand is how a wrong constant gets
published — and a wrong constant in a test file is worse than a missing
one, because it sends an implementer looking for a bug in correct code.
The implementation lane generates the table from the reference binary
and commits it as its first step.

## Status

| item | implemented |
| --- | --- |
| the twenty-seven `XXH*` size and prime constants | yes — they are constants |
| `xxh32.Xxh32State`, `xxh64.Xxh64State`, `xxh3.Xxh128`, `.Xxh3State`, `xxherr.XxhError` | declared |
| `xxh32.mask`, `.rotl`, `.round`, `.avalanche` | no |
| `xxh32.start`, `.feed_byte`, `.finish` and the six accessors | no |
| `xxh32.byte_of` | no |
| `xxh64.rotl`, `.round`, `.merge_lane`, `.avalanche` | no |
| `xxh64.start`, `.feed_byte`, `.finish` and the six accessors | no |
| `xxh64.byte_of`, `.low32` | no |
| `xxh3.digest128`, `.digest_low`, `.digest_high`, `.digest_byte` | no |
| `xxh3.acc_start`, `.acc_lane`, `.with_acc_lane`, `.stripe`, `.total` | no |
| `xxh3.path_for`, `.check_secret` | no |
| `xxhbytes`'s four XXH32 and four XXH64 functions | no |
| `xxhbytes`'s six XXH3 functions | no |
| `xxhbytes.text32`, `.text64`, `.text128` | no |
| `xxherr.describe`, `XxhError.message` | no |
| **XXH3's streaming state** | **outside, and named above** |

## Licence

Apache-2.0.  See `LICENSE`.
