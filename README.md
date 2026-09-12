# xxhash-nv

xxHash is a family of fast, non-cryptographic hash functions. The reference
implementation and the written specification are at
[Cyan4973/xxHash](https://github.com/Cyan4973/xxHash). This package brings
XXH32, XXH64 and XXH3 to novo-lang, as three separate algorithms with three
separate entry points.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What xxHash is

A non-cryptographic hash turns bytes into a fixed-width number, quickly. It
detects accidental corruption very well: a failing disk, a truncated download, a
bit flipped in transit. It offers no defence against a deliberate one — an
adversary who can choose the bytes can produce a collision in seconds. Every
format that uses xxHash says exactly that much, and so does this page.

xxHash is not a cyclic redundancy check. A CRC is a different family with a
different purpose: it is designed to catch burst errors on a wire and has
provable detection properties for them. Use a CRC package for that; use this one
for the formats that specify xxHash.

The three algorithms are genuinely different, not three widths of one.

| | XXH32 | XXH64 | XXH3 |
| --- | --- | --- | --- |
| Digest | 4 bytes | 8 bytes | 8 or 16 bytes |
| Stripe | 16 bytes, 4 lanes of 4 | 32 bytes, 4 lanes of 8 | 64 bytes, 8 lanes |
| Round | accumulate, rotate 13, multiply | accumulate, rotate 31, multiply | — |
| Merge | XOR the four rotated lanes | applies the round again to each lane, then multiplies by prime 1 and adds prime 4 | — |
| Tail | 4 bytes, then 1 | 8 bytes, then 4, then 1 | — |
| Final mixing | shifts 15, 13, 16 | shifts 33, 29, 32 | — |
| Primes | five 32-bit numbers | five different 64-bit numbers | a 192-byte secret |

A port that widened XXH32 into XXH64 passes nothing.

XXH3 shares none of the above. It has a 192-byte *secret*, eight accumulator
lanes designed for vector instructions, and **four separate code paths chosen by
input length**: at or below 16 bytes, 17 to 128, 129 to 240, and above. Those
bounds are published as constants, because they are the first thing an
implementer lines up against the reference and because a measurement that covers
three of the four reports the wrong thing.

**XXH3 has no specification document.** xxHash publishes an `xxhash_spec.md`
covering XXH32 and XXH64. XXH3 is defined by the reference source, with the
guarantee that its output has been stable since version 0.8.0. For XXH3 the
reference implementation is not a second opinion; it is the definition.

## Install

```
novo pkg add xxhash-nv
```

## Example

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

Build and test with:

```
novo pkg build
novo test
```

Today `novo test` fails on purpose: every assertion reaches
`not implemented: xxhash-nv.<module>.<fn>`.

## What the package contains

| Module | Contents | Builds for a microcontroller |
| --- | --- | --- |
| `xxh32` | XXH32 whole: the state as a plain value, the round, the final mixing, `start`, `feed_byte` and `finish`, the five primes, and `byte_of`. | yes |
| `xxh64` | XXH64 whole, the same way, plus `low32`. | yes |
| `xxh3` | XXH3's constants, its four length paths, the 128-bit digest type, the accumulator bank, and the secret check. | no |
| `xxhbytes` | The bridge: `Bytes` in, digests and printable text out, for all four widths. | no |
| `xxherr` | The error type `XxhError` and its message. | no |

No function in this package draws randomness, reads a clock or performs input or
output. The seed, the secret and the bytes all arrive as arguments. The package
has no dependencies.

## How to choose an entry point

**There is no `hash(data, width)` and there will not be one.** `hash32`,
`hash64`, `hash3_64` and `hash3_128` are four names typed on purpose.

The reason is that two formats a program may read on the same afternoon each
store a four-byte checksum over the same content, and they are different
numbers. A Zstandard frame's content checksum is the low half of an XXH64. An
LZ4 frame's content checksum is an XXH32. A width parameter would make computing
the wrong one a typo; four names make it a decision. The same applies to XXH3,
which a caller asking for "64 bits" might be given, and whose digest no
Zstandard reader will ever accept.

**Use `xxhbytes.hash32` or `.hash64` for a one-shot over a buffer.** The `_range`
forms hash a window of a scratch buffer without slicing it, which is the
difference between one allocation per block and one in total.

**Use `xxh32.start` or `xxh64.start` with `xxhbytes.feed32` or `.feed64` to
stream.** `finish` is a pure read: it does not consume the state, so a caller
can report a running hash and keep feeding. That is what a compressor emitting
both a per-block checksum and a whole-content checksum has to do.

**Use `xxhbytes.hash3_64` and `.hash3_128` for XXH3.** The seeded and
custom-secret forms are beside them.

**Firmware calls `xxh32` or `xxh64` directly.** See "Running on a
microcontroller".

## The rules a user needs

1. **`Int` is 64-bit and signed, so a finished XXH64 is very often negative.**
   Printing one with `${h}` gives a minus sign and a value no other tool has
   produced. `xxhbytes.text32`, `.text64` and `.text128` print what `xxhsum`
   prints.
2. **Three of XXH64's five primes have their high bit set**, so the source
   writes them as negative decimals with the hexadecimal in a comment.
3. **XXH32's words are 32 bits inside a 64-bit `Int`, so every product and
   rotation comes back through `xxh32.mask`.** A missing mask produces plausible
   values that diverge later. `mask` is public and the tests assert it at the
   round function.
4. **`xxhbytes.text128` puts the high half first**, which is the opposite of the
   canonical byte order `xxhbytes.digest128_into` writes. Every 128-bit hash's
   tooling has this disagreement. Both are published so nobody has to guess
   which they are looking at.
5. **A custom XXH3 secret must be at least 136 bytes.** XXH3's long-input path
   reads the secret at offsets derived from the stripe counter, so a shorter one
   would be read past its end. The reference refuses below 136 and so does this.
   `XxhSecretTooShort(got, least)`.
6. **A seed and a custom secret cannot both be given.** They are the same knob:
   XXH3 derives a secret *from* a seed. The reference quietly ignores one of
   them; this refuses with `XxhSeedAndSecretBothGiven`, because the two choices
   produce different digests and the caller has not said which they meant.
7. **An output buffer smaller than the digest is `XxhOutputTooSmall(want,
   have)`.**
8. **`xxh64.low32` and `xxh32.byte_of` are published** because the truncation a
   Zstandard trailer stores and the byte an LZ4 header stores are one line each,
   and both are lines that every consumer rediscovers and half of them get
   wrong.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device with no
heap allocator, and the registry checks that claim by compiling a probe program
for a Cortex-M4. Here the claim covers `xxh32` and `xxh64`, and what it gives a
device is two whole hashes: `start`, `feed_byte`, `finish`.

Those modules take no `Bytes`, no `Str` and no list. A state is four
accumulators and a stripe buffer inside a plain value struct: 56 bytes of the
caller's frame for XXH32, 88 for XXH64. `tests/embedded_probe.nv` builds both
for the device.

The device is a real case. The LZ4 frame format checks three XXH32 values — a
header byte, a per-block checksum, a content checksum — and a bootloader that
can decompress but cannot check the checksum has verified nothing. Zstandard is
the same argument at XXH64.

`xxh3` is deliberately outside the claim. Its accumulator layout exists to be
eight lanes of vector arithmetic. On a Cortex-M4 the scalar path is slower than
XXH32 and costs some 256 bytes more of state and secret. A device that wants a
checksum wants `xxh32`.

## What is not included

- **XXH3's streaming state.** The accumulator bank is declared so the shape is
  public, but the streaming form is not here: it also needs a 256-byte input
  buffer and a derived secret, and its block-accumulation rule is a second
  algorithm's worth of detail. The one-shot forms cover every published consumer
  today. This is the first thing to add when one asks.
- **XXH128's comparison and canonical-form helpers.** `Xxh128` is two integers
  and comparing two integers needs no helper. The canonical bytes are
  `xxhbytes.digest128_into`.
- **Vector instructions.** The reference gets its speed from AVX2 and NEON
  paths. This is a scalar port and will be slower. Nothing in the interface
  would change if a vectorised implementation arrived.
- **XXH1 and XXH0**, superseded by the algorithm's own author.
- **A cyclic redundancy check.** Different family, different promise.

## Related packages

[zstd-nv](https://novo-lang.org/packages/zstd-nv) and
[lz4-nv](https://novo-lang.org/packages/lz4-nv) each carry a private copy of the
hash their format needs, and each can call this package instead.

| zstd-nv's `zstdxxh` | Here |
| --- | --- |
| `ZstXxh64` | `xxh64.Xxh64State` |
| `start(seed)` | `xxh64.start(seed)` |
| `feed(h, chunk)` | `xxhbytes.feed64(h, chunk)` |
| `finish(h)` | `xxh64.finish(h)` |
| `hash(data)` | `xxhbytes.hash64(data, 0)` |
| `truncate32(h)` | `xxh64.low32(h)` |
| `as_unsigned_text(h)` | `xxhbytes.text64(h)` |
| `stripe_bytes()` | `xxh64.XXH64_STRIPE_BYTES` |

| lz4-nv's `lz4xxh` | Here |
| --- | --- |
| `Lz4Xxh32` | `xxh32.Xxh32State` |
| `start(seed)` | `xxh32.start(seed)` |
| `feed(h, chunk)` | `xxhbytes.feed32(h, chunk)` |
| `finish(h)` | `xxh32.finish(h)` |
| `hash(data)` | `xxhbytes.hash32(data, 0)` |
| `header_byte(h)` | `xxh32.byte_of(h, 1)` |
| `as_unsigned_text(h)` | `xxhbytes.text32(h)` |
| `stripe_bytes()` | `xxh32.XXH32_STRIPE_BYTES` |

## Test vectors

`Cyan4973/xxHash` is the oracle. For XXH32 and XXH64 its `xxhash_spec.md` is the
written definition and the source is the second check. For XXH3 the source is
the definition.

The tests carry:

- **The digest of the empty input at all four widths**: `02cc5d05`,
  `ef46db3751d8e999`, `2d06800538d394c2` and
  `99aa06d3014798d86001c324468d497f`. It is the value every port prints first,
  and the one every implementation agrees on before it agrees on anything else.
- Every structural property that does not need a table: that chunking the input
  across a stripe boundary changes nothing, that the seed changes the answer,
  that each of XXH3's four length paths is exercised, the two canonical byte
  orders, and the exact calls the two consumer packages make.

The reference's full sanity table is not transcribed here. It hashes prefixes of
a 2367-byte buffer produced by its own generator, at several seeds, and runs to
over a hundred values. A wrong constant in a test file is worse than a missing
one, because it sends an implementer looking for a bug in correct code. The
implementation generates that table from the reference binary and commits it as
its first step.

## Implementation status

| Item | Implemented |
| --- | --- |
| The twenty-seven size and prime constants | yes (they are constants) |
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

## Licence

Apache-2.0. See `LICENSE`.
