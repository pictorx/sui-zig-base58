# Base58 Decoding

This implementation is a Zig port of the fast base58 decoder from
[Firedancer](https://github.com/firedancer-io/firedancer), found in
`src/ballet/base58/fd_base58.c`. All credit for the algorithm and
optimizations belongs to the Firedancer authors.

---

## Overview

Decoding is the exact reverse of the encoding pipeline. A base58 string
is treated as a big number written in base 58, and we convert it back to
the original bytes.

```
base 58           base 58⁵         base 2³²        [N]u8
(encoded   ──(1)──▶ intermediate ──(2)──▶  limbs  ──(3)──▶ bytes
 string)            (u64 values)   (u32 words)
```

---

## Stage 1 — Base58 string to base 58⁵ intermediate

### 1a. Character parsing

Each character is mapped from the alphabet to a raw digit (0–57) using
an inverse lookup table. Invalid characters immediately return an error.

Leading `'1'` characters are counted separately — they represent leading
zero bytes and are not part of the numeric value.

### 1b. Right-alignment

The raw digits are placed right-aligned into a fixed-size buffer of
`intermediateSize(N) * 5` slots, with zeros filling the left side. This
mirrors the natural position of the digits in the encoding.

### 1c. Grouping into base 58⁵

Each group of 5 consecutive raw digits is combined into a single `u64`
intermediate value using Horner's rule expanded:

```
intermediate[i] = d4 * 58⁴ + d3 * 58³ + d2 * 58² + d1 * 58 + d0
               = d4 * 11,316,496 + d3 * 195,112 + d2 * 3,364 + d1 * 58 + d0
```

**SIMD optimization:** all `inter_sz` groups are computed simultaneously
using `@Vector(inter_sz, u64)`, mirroring the vectorized decomposition
in the encoder's `intermediateToRaw`.

---

## Stage 2 — Base 58⁵ intermediate to 32-bit limbs

This is the inverse matrix multiply using `dec_table`:

```
binary[k] += intermediate[j] * dec_table[j][k]
```

where `dec_table[j][k]` is the `k`-th base-2³² digit of `58^(5*(inter_sz−1−j))`.
Each table row holds the base-2³² representation of the positional value
of one intermediate slot.

```zig
fn makeDecTable(comptime N: usize) [intermediateSize(N)][N/4]u32 {
    // dec_table[j][k] = k-th base-2^32 digit of 58^(5*(inter_sz-1-j))
}
```

**Why u128 accumulators?** Each product is at most `(58⁵−1) × (2³²−1) < 2⁶²`.
Summing up to 18 such products reaches `~2⁶⁶`, overflowing `u64`. The
accumulator must be `u128`. This prevents SIMD here — unlike the encoder's
matrix multiply, there is no 128-bit SIMD lane mapping.

After accumulating, a carry pass normalizes each limb to 32 bits:

```
binary[k-1] += binary[k] >> 32
binary[k]   &= 0xFFFF_FFFF
```

If `binary[0]` still has bits above bit 31, the input encodes a value
larger than `N` bytes — this is an overflow error.

---

## Stage 3 — Limbs to bytes

The 32-bit limbs are converted back to a big-endian byte array using the
same `@byteSwap` on a `@Vector` as the encoder uses in reverse:

```zig
const bytes: [32]u8 = @bitCast(@byteSwap(@as(@Vector(8, u32), @bitCast(limbs))));
```

---

## Leading Zeros Validation

After decoding, the number of leading zero bytes in the output must
exactly match the number of leading `'1'` characters in the input.
A mismatch means the encoding was non-canonical (too many or too few
`'1'`s), and the function returns a decode error.

---

## Error Cases

| Error              | Cause                                              |
|--------------------|----------------------------------------------------|
| `InvalidCharacter` | A character is not in the base58 alphabet          |
| `Decode`           | Input too long, value overflows N bytes, or leading zeros mismatch |
| `NoSpaceLeft`      | Output buffer smaller than N bytes                 |
