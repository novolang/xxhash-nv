# Changelog

All notable changes to xxhash-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-11-01

XXH32, XXH64 and XXH3 at 64 and 128 bits, with a body for every function.

### Changed

- **Breaking.** The `out` parameter of `digest32_into`, `digest64_into`,
  `digest128_into` and `hash3_128_with_secret_into` is `var out: Bytes`. A
  `Bytes` is written only through a `var` name, so the buffer a caller handed
  in could not be filled under the earlier signature. A call that passes a
  fresh `bytes.zeros(n)` or a `var` buffer is unchanged.
- Every function of `xxh32` and `xxh64` carries `@tier(embedded)`.
- `stability` is `experimental`.

### Added

- Bodies for every function. The one-shots take whole stripes straight from the
  buffer, and the range forms clamp a range that runs off the buffer to the
  bytes that exist.
- `tests/xxh_oracle_tests.nv`: 44 input lengths against the reference library
  at every width, seeded and over a secret; `tests/xxh_edges_tests.nv`;
  `tests/coverage.sh`.

This release requires novo 0.23.0 or later.

## 0.0.2 — 2026-09-12

### Changed — The README is rewritten in plain technical-writer prose; no signature changed.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `xxh32` — `Xxh32State`, the five 32-bit primes, the mask, the left
  rotation, the round, the avalanche, and a whole streaming hash in
  `start` / `feed_byte` / `finish` with no `Bytes` in the module;
  `byte_of` for the byte an LZ4 frame header stores.
- `xxh64` — the same surface at a 32-byte stripe and 64-bit primes,
  with `merge_lane` published separately because XXH64 applies the
  round a second time on the way out and XXH32 does not, and `low32`
  for the truncation a Zstandard frame trailer stores.
- `xxh3` — the secret sizes, the four length-path boundaries, the long
  path's geometry, `Xxh128` as two integers with its canonical byte
  order, the accumulator bank, `path_for` and `check_secret`.
- `xxhbytes` — the bridge: feed and one-shot at every width, the range
  forms a frame reader needs, the canonical digest writers, and the
  three text forms that answer what `xxhsum` prints.
- `xxherr` — one short error type, every arm XXH3's or a caller's
  buffer.

### Known

- **The load-bearing interface is that there is no `hash(data,
  width)`.**  A Zstandard trailer is XXH64's low half and an LZ4
  trailer is XXH32 — two four-byte checksums over the same content that
  are different numbers — so the width is a name a caller types rather
  than a parameter they can get wrong.
- **`finish` is a pure read**, so a compressor emitting both a
  per-block and a whole-content checksum does not have to duplicate a
  state.
- **The device claim covers `xxh32` and `xxh64`** and covers two WHOLE
  hashes; `tests/embedded_probe.nv` builds them for a Cortex-M4.  A
  bootloader that can decompress and cannot check the checksum has
  verified nothing.
- **`xxh3` is deliberately outside the claim**: its accumulator layout
  exists to be SIMD, and on a Cortex-M4 the scalar path is slower than
  XXH32 at ~256 bytes more state.
- **XXH3 has no specification document** — the reference source is the
  definition, with output stable since v0.8.0.
- **A finished XXH64 is often a negative `Int`**, so `text64` exists
  and the README says why `${h}` is not it.
- **zstd-nv's `zstdxxh` and lz4-nv's `lz4xxh` both delete**: the README
  maps every function in each onto a call here.
- xxHash is **not cryptographic and not a CRC**, and the README says
  which promise each family makes.
- XXH3's streaming state, the SIMD paths, XXH1 and XXH0 are outside,
  each with a reason.
