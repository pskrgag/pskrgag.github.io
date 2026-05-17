+++
title = "Parsing and formatting UUIDv4 with SIMD"
description = "Notes for future me"
date = "2026-05-16"
+++

## Intro

Sometimes I spend evenings solving [highload.fun](https://highload.fun/home) tasks. Just to learn some CPU performance things in practice rather than from books. So I decided to write down some notes on SIMD UUIDv4 parsing for the [Sort UUIDs](https://highload.fun/challenges/compute/sort_uuids/solve/RUST) task.

The next sections assume at least basic x86 SIMD knowledge.

## Parsing

UUIDv4 has a very simple format:

```
<8 hex digits>-<4 hex digits>-<4 hex digits>-<4 hex digits>-<12 hex digits>
```

37 bytes in total. My machine only has AVX2, which gives access to 32-byte registers. Anyway, it is enough for parsing.

### Parsing algorithm

Let's take a random UUID and parse it.

```
fb3115c3-49af-4617-b86a-14c81e293da4
```

Firstly, let's load first 32 bytes into avx2 register:

```
['f', 'b', '3', '1', '1', '5', 'c', '3', '-', '4', '9', 'a', 'f', '-', '4', '6', '1', '7', '-', 'b', '8', '6', 'a', '-', '1', '4', 'c', '8', '1', 'e', '2', '9']
```

Then we need to get rid of dashes, since they are not part of the number. First thing that comes to mind is to use `_mm256_shuffle_epi8` to move bytes right to dashes positions. However, there is a problem -- `_mm256_shuffle_epi8` cannot do cross-lane shuffle. Why is it a problem?

Our original layout is following

```
['f', 'b', '3', '1', '1', '5', 'c', '3', '-', '4', '9', 'a', 'f', '-', '4', '6', // lane1
 '1', '7', '-', 'b', '8', '6', 'a', '-', '1', '4', 'c', '8', '1', 'e', '2', '9'] // lane2
```

But we want

```
['f', 'b', '3', '1', '1', '5', 'c', '3', '4', '9', 'a', 'f', '4', '6', '1', '7' // lane1
 'b', '8', '6', 'a', '1', '4', 'c', '8', '1', 'e', '2', '9', ...]               // lane2
```

In case you didn't notice: '1' and '7' are moved across lane. The only thing we can do is to keep all chars on their lanes and have 2 unused bytes at the end of each lane.

```
['f', 'b', '3', '1', '1', '5', 'c', '3', '4', '9', 'a', 'f', '4', '6', '\0', '\0', // lane1
 '1', '7', 'b', '8', '6', 'a', '1', '4', 'c', '8', '1', 'e', '2', '9', '\0', '\0'] // lane2
```

This is done with `_mm256_shuffle_epi8` and following shuffle mask:

```rust
let shuffle_mask: [u8; 32] = [
    0, 1, 2, 3, 4, 5, 6, 7, 9, 10, 11, 12, 14, 15, 1 << 7, 1 << 7,
    0, 1, 3, 4, 5, 6, 8, 9, 10, 11, 12, 13, 14, 15, 1 << 7, 1 << 7,
];
```

`1 << 7` has a special meaning for `_mm256_shuffle_epi8`: destination byte is zeroed.

Note that indexes in second lane are relative to this lane, not to the whole 32-byte register. This is the thing I forgot multiple times. Avx2 is 256-bit, but some operations are still two 128-bit operations in disguise.

### Hex to nibbles

After removing dashes each byte is still ascii character. We need to convert it to number in range `[0, 15]`. For scalar code there is a nice branchless trick:

```rust
let x = (c & 0x0f) + (c >> 6) * 9;
```

Why does it work?

For digits:

```
'0' = 0x30
'9' = 0x39

c & 0x0f = 0..9
c >> 6   = 0
```

For lowercase letters:

```
'a' = 0x61
'f' = 0x66

c & 0x0f = 1..6
c >> 6   = 1
```

So for letters we add 9 and get `10..15`.

Unfortunately avx2 does not have `u8 * u8 -> u8` instruction. However SIMD version is still simple:

```rust
let sub = _mm256_subs_epu8(shuffled, _mm256_set1_epi8(b'0' as _));
let norm = _mm256_and_si256(
    _mm256_set1_epi8(39 as _),
    _mm256_cmpgt_epi8(shuffled, _mm256_set1_epi8(b'9' as _)),
);
let num = _mm256_subs_epu8(sub, norm);
```

For digits `norm` is zero, so we just subtract `'0'`.

For letters we subtract `'0'` and then subtract 39 more:

```
'a' - '0' = 49
49 - 39 = 10
```

Now register contains nibbles:

```
[15, 11, 3, 1, 1, 5, 12, 3, 4, 9, 10, 15, 4, 6, 0, 0, ...]
```

### Packing nibbles into bytes

We want to convert

```
[0xf, 0xb, 0x3, 0x1, ...]
```

into

```
[0xfb, 0x31, ...]
```

The idea is:

```
byte = (even_nibble << 4) | odd_nibble
```

There is no direct shift for `u8` lanes in avx2. But since nibbles are small, we can shift as `u16` and then shuffle:

```rust
let high = _mm256_shuffle_epi8(
    _mm256_slli_epi16(num, 4),
    _mm256_loadu_si256(shuffle_even.as_ptr() as _),
);
let low = _mm256_shuffle_epi8(num, _mm256_loadu_si256(shuffle_odd.as_ptr() as _));
let num = _mm256_or_si256(high, low);
```

Masks are:

```rust
let shuffle_even: [u8; 32] = [
    0, 2, 4, 6, 8, 10, 12, 14, 1 << 7, 1 << 7, 1 << 7, 1 << 7, 1 << 7, 1 << 7, 1 << 7, 1 << 7,
    0, 2, 4, 6, 8, 10, 12, 14, 1 << 7, 1 << 7, 1 << 7, 1 << 7, 1 << 7, 1 << 7, 1 << 7, 1 << 7,
];

let shuffle_odd: [u8; 32] = [
    1, 3, 5, 7, 9, 11, 13, 15, 1 << 7, 1 << 7, 1 << 7, 1 << 7, 1 << 7, 1 << 7, 1 << 7, 1 << 7,
    1, 3, 5, 7, 9, 11, 13, 15, 1 << 7, 1 << 7, 1 << 7, 1 << 7, 1 << 7, 1 << 7, 1 << 7, 1 << 7,
];
```

What about the tail `3da4`? Initially I parsed it with scalar loop. It worked, but it was slower than I expected. Current version inserts these 4 bytes back into the vector before ascii-to-nibble conversion:

```rust
let shuffled = _mm256_insert_epi16(
    shuffled,
    (ptr.add(i * 37 + 32) as *const u16).read() as _,
    15,
);
let shuffled = _mm256_insert_epi16(
    shuffled,
    (ptr.add(i * 37 + 34) as *const u16).read() as _,
    7,
);
```

This looks a bit weird, but the idea is simple: we have two zero holes after shuffle, one in each lane. Tail bytes are inserted into those holes, so the same SIMD conversion and packing code handles all 32 hex digits. No scalar tail loop.

After this step each lane contains 8 useful bytes:

```
lane1: fb 31 15 c3 49 af 46 a4 ...
lane2: 17 b8 6a 14 c8 1e 29 3d ...
```

### Combining lanes into `u128`

Next problem is to construct one `u128` out of two lanes. I used transmute to get lower 128-bit lane as `u128`:

```rust
fn m128i_to_u128(vector: __m128i) -> u128 {
    unsafe { std::mem::transmute(vector) }
}
```

Then:

```rust
let upper = m128i_to_u128(_mm256_castsi256_si128(num));
let low = m128i_to_u128(_mm256_extracti128_si256(num, 1));
```

Values in lanes are in byte order, but integer on x86 is little-endian, so byte swap is required. Also note that last two bytes are in "wrong" lanes because of tail insertion. Current code handles this with a couple of masks and shifts:

```rust
let result = (upper.swap_bytes() & !(0xFFu128 << (8 * 8)))
    | (low.swap_bytes() >> (64 - 8))
    | ((upper.swap_bytes() >> (8 * 8)) & 0xFF);
```

This expression is ugly, but it removed scalar tail handling from hot loop. In my measurements it was worth it.

And that's it. UUID string is now one `u128`.

## Formatting

Formatting is the same thing in reverse. Given one `u128`, convert it to bytes, split every byte into two nibbles and convert nibbles to ascii characters.

For one byte:

```
0xfb -> 0x0f 0x0b -> 'f' 'b'
```

The nice SIMD trick here is `_mm256_unpacklo_epi8`. Suppose we have two vectors:

```
hi = [h0, h1, h2, h3, ...]
lo = [l0, l1, l2, l3, ...]
```

Then `_mm256_unpacklo_epi8(hi, lo)` zips them:

```
[h0, l0, h1, l1, h2, l2, ...]
```

Which is exactly what hexadecimal string wants.

High and low nibbles are computed like:

```rust
let hi = _mm256_and_si256(_mm256_set1_epi8(0xf), _mm256_srli_epi16(chunk, 4));
let low = _mm256_and_si256(_mm256_set1_epi8(0xf), chunk);
```

Again, there is no `u8` shift, so `u16` shift + mask is used.

Then:

```rust
let lo_ascii = _mm256_unpacklo_epi8(hi, low);
let hi_ascii = _mm256_unpackhi_epi8(hi, low);
let res = _mm256_inserti128_si256(lo_ascii, _mm256_castsi256_si128(hi_ascii), 1);
```

At this point `res` contains 32 bytes, each byte is value in range `[0, 15]`.

To convert nibbles to ascii:

```rust
let norm = _mm256_and_si256(
    _mm256_set1_epi8(39 as _),
    _mm256_cmpgt_epi8(res, _mm256_set1_epi8(9 as _)),
);

let add = _mm256_add_epi8(res, _mm256_set1_epi8(b'0' as _));
let num = _mm256_add_epi8(add, norm);
```

For numbers `0..9`, add `'0'`. For letters `10..15`, add `'0' + 39`, which is the same as `'a' - 10`.

### Dashes

UUID string has dashes at fixed positions:

```
8, 13, 18, 23
```

So I use another shuffle to create holes and then OR dashes into those holes.

There was one more cross-lane problem here. Output position `16` and `17` need input bytes `14` and `15`, but output position `16` belongs to second 128-bit lane of avx2 register. `_mm256_shuffle_epi8` cannot take bytes from previous lane.

I solved it with a small fixup:

```rust
_mm256_storeu_si256(to as _, dashes);

(to.add(32) as *mut u32).write(_mm256_extract_epi32(num, 7) as _);
(to.add(16) as *mut u16).write(_mm256_extract_epi16(num, 7) as _);
to.add(36).write(b'\n');
```

Not the most beautiful code in the world, but it works.
