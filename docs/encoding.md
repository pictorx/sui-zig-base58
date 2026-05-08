# Base58 Encoding

This implementation is a Zig port of the fast base58 encoder from
[Firedancer](https://github.com/firedancer-io/firedancer), found in
`src/ballet/base58/fd_base58.c`. All credit for the algorithm and
optimizations belongs to the Firedancer authors.

---

## What is Base58?

A Base58-encoded string is simply a big number written in base 58. The
alphabet skips four visually ambiguous characters (`0`, `O`, `I`, `l`),
leaving 58 printable digits:

```
123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz
 ^                                                         ^
 digit 0                                               digit 57
```

So encoding `[N]u8` means: interpret those bytes as one large integer,
then write that integer in base 58.

---

## The Three-Stage Pipeline

Directly converting a 256-bit number to base 58 one digit at a time
would require repeated big-number division. Instead, Firedancer uses two
intermediate bases that make the work divisible into fast fixed-size
chunks.

```
[N]u8  ──(1)──▶  base 2³²  ──(2)──▶  base 58⁵  ──(3)──▶  base 58
       limbs     (8 or 16   intermediate   (45 or 90
                  u32 words)  u64 values)   u8 digits)
```

### Stage 1 — Bytes to 32-bit limbs

The input bytes are reinterpreted as an array of big-endian `u32` words.
A 32-byte pubkey becomes 8 limbs; a 64-byte signature becomes 16 limbs.

```
[0x01, 0x02, 0x03, 0x04, ...]  →  [0x01020304, ...]
```

In Zig this is a single `@byteSwap` on a `@Vector`:

```zig
fn pubkeyLimbs(src: [32]u8) [8]u32 {
    const limbs: @Vector(8, u32) = @bitCast(src);
    return @bitCast(@byteSwap(limbs));
}
```

### Stage 2 — Limbs to base 58⁵ intermediate

`58⁵ = 656,356,768`, which is less than `2³²`, so each intermediate
value fits in a `u64`. The full value expressed in this base uses 9
digits for 32 bytes (or 18 for 64 bytes).

The conversion is a matrix multiply:

```
intermediate[j] += limbs[i] * enc_table[i][j]
```

where `enc_table[i][j]` is the `j`-th base-58⁵ digit of `2^(32*(N/4−1−i))`.
In other words, each table row holds the base-58⁵ representation of the
positional value of one 32-bit limb.

The table is generated at comptime:

```zig
fn makeEncTable(comptime N: usize) [N/4][intermediateSize(N)-1]u32 {
    // enc_table[i][j] = j-th base-58^5 digit of 2^(32*(binary_sz-1-i))
}
```

After the multiply, a carry pass normalizes all intermediate values to
be less than `58⁵`.

**SIMD optimization:** the inner loop over `j` is vectorized with
`@Vector`, broadcasting each `limbs[i]` scalar and multiplying it by
the whole table row at once.

### Stage 3 — Intermediate to base 58

Each intermediate value (< 58⁵) decomposes into exactly 5 base-58
digits by successive division:

```
d4 = v / 58⁴           (= v / 11,316,496)
d3 = v / 58³ mod 58     (= v /    195,112  mod 58)
d2 = v / 58² mod 58
d1 = v / 58  mod 58
d0 = v       mod 58
```

This is also vectorized: all `inter_sz` intermediate values are
processed simultaneously using `@Vector(inter_sz, u32)`.

---

## Leading Zeros

Each leading zero byte in the input maps to one `'1'` character in the
output. This is handled separately:

1. Count leading zero bytes: `in_leading_zero`.
2. After computing the raw digit array, count its leading zero digits:
   `raw_leading_zero`.
3. Skip `raw_leading_zero − in_leading_zero` raw digits (the natural
   leading zeros of the numeric value).
4. Prepend `in_leading_zero` copies of `'1'`.

---

## Output Buffer Size

The maximum encoded length for `N` bytes is `intermediateSize(N) * 5`:

| Input  | `intermediateSize` | Max encoded chars |
|--------|--------------------|-------------------|
| 32 B   | 9                  | 45                |
| 64 B   | 18                 | 90                |

The actual output is often shorter due to leading zeros being stripped.
