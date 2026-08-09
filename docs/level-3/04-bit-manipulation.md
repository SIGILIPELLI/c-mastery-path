# 04 · Bit Manipulation

Every value in C is, underneath the type system, just a run of bits in
memory. Most code never needs to think below the level of `int` or
`struct` — but flags, protocol headers, hashing, and hardware registers all
live at the bit level, and C is one of the few languages that lets you work
there directly with plain operators instead of a library. This module
covers the operators, the classic bit tricks built from them, and the traps
that make bit manipulation one of the easiest places to introduce undefined
behavior without realizing it.

## The operators

C has six bitwise operators, all working on the binary representation of
integers:

| Operator | Name | Example (`x = 0b0110`) |
|---|---|---|
| `&` | AND | `x & 0b0011` → `0b0010` |
| `\|` | OR | `x \| 0b0011` → `0b0111` |
| `^` | XOR | `x ^ 0b0011` → `0b0101` |
| `~` | NOT (complement) | `~x` → flips every bit |
| `<<` | left shift | `x << 1` → `0b1100` |
| `>>` | right shift | `x >> 1` → `0b0011` |

These are entirely different from `&&`, `||`, and `!` (the logical
operators) — `&` operates bit-by-bit on the whole value, `&&` treats each
operand as a single true/false and short-circuits. Mixing them up (`if (a &
b)` when you meant `if (a && b)`) compiles without a single warning and is
one of the most common bit-manipulation bugs in practice.

## Bit flags: set, clear, toggle, check

The most common real-world use of bitwise operators is packing several
boolean flags into one integer — one bit per flag, instead of one whole
`int` or `bool` per flag:

```c
// bits1.c
#include <stdio.h>

#define BIT(n) (1u << (n))

int main(void) {
    unsigned int flags = 0;

    flags |= BIT(1);   // set bit 1
    flags |= BIT(3);   // set bit 3
    printf("after setting bits 1,3: 0x%02X\n", flags);

    flags &= ~BIT(1);  // clear bit 1
    printf("after clearing bit 1:   0x%02X\n", flags);

    flags ^= BIT(3);   // toggle bit 3 (was set, becomes clear)
    printf("after toggling bit 3:   0x%02X\n", flags);

    flags ^= BIT(3);   // toggle again (clear -> set)
    printf("after toggling bit 3 again: 0x%02X\n", flags);

    int is_set = (flags & BIT(3)) != 0;
    printf("is bit 3 set? %s\n", is_set ? "yes" : "no");

    return 0;
}
```

```bash
gcc -Wall -Wextra -g -O0 -fsanitize=address,undefined -o bits1 bits1.c
./bits1
```

```text
after setting bits 1,3: 0x0A
after clearing bit 1:   0x08
after toggling bit 3:   0x00
after toggling bit 3 again: 0x08
is bit 3 set? yes
```

Four operations, four idioms worth memorizing exactly as written:

- **Set bit n**: `x |= BIT(n)` — OR-ing with a 1 forces that bit on and
  leaves every other bit untouched (OR-ing with 0 changes nothing).
- **Clear bit n**: `x &= ~BIT(n)` — `~BIT(n)` is all 1s except a single 0
  at position n, so AND-ing forces that bit off and leaves the rest alone.
- **Toggle bit n**: `x ^= BIT(n)` — XOR-ing with 1 flips a bit; XOR-ing
  with 0 leaves it unchanged.
- **Check bit n**: `(x & BIT(n)) != 0` — AND-ing with a single 1 isolates
  that bit; every other bit becomes 0, so the whole expression is nonzero
  only if bit n was set.

## Shifting, and where it gets dangerous

`<<` and `>>` move bits left or right, filling vacated positions with 0 —
*except* right-shifting a **signed negative** value, which is
implementation-defined and on essentially every real compiler (including
gcc and clang) performs an **arithmetic shift**: it fills with copies of
the sign bit instead of 0, so a negative number shifted right stays
negative.

```c
// bits2.c -- shifting, sign extension, and Brian Kernighan's popcount
#include <stdio.h>

int count_bits_naive(unsigned int n) {
    int count = 0;
    while (n) {
        count += (n & 1);
        n >>= 1;
    }
    return count;
}

int count_bits_kernighan(unsigned int n) {
    int count = 0;
    while (n) {
        n &= (n - 1);   // clears the lowest set bit each time
        count++;
    }
    return count;
}

int main(void) {
    unsigned int x = 0b10110100;
    printf("x = 0x%02X, naive popcount = %d, kernighan popcount = %d\n",
           x, count_bits_naive(x), count_bits_kernighan(x));

    int signed_val = -8;              // 0xFFFFFFF8 in two's complement
    printf("\n-8 >> 1 (signed, arithmetic shift): %d\n", signed_val >> 1);

    unsigned int unsigned_val = (unsigned int)signed_val;
    printf("-8 reinterpreted as unsigned >> 1 (logical shift): %u\n",
           unsigned_val >> 1);

    unsigned int one = 1;
    printf("\n1u << 31 = %u\n", one << 31);
    // 1u << 32 would be undefined behavior on a 32-bit unsigned int -- not tested here

    return 0;
}
```

```text
x = 0xB4, naive popcount = 4, kernighan popcount = 4

-8 >> 1 (signed, arithmetic shift): -4
-8 reinterpreted as unsigned >> 1 (logical shift): 2147483644

1u << 31 = 2147483648
```

`-8 >> 1` gives `-4` (an arithmetic shift preserves sign, so this matches
integer division by 2), while the *same bit pattern* reinterpreted as
`unsigned int` and shifted gives `2147483644` — an enormous positive number,
because a logical shift fills with 0 regardless of what the top bit used to
mean. Same bits, same operator, opposite-looking results, purely because of
signedness. This is why bit-manipulation code should almost always use
unsigned types (`unsigned int`, `uint32_t`) unless you specifically want
sign-preserving arithmetic shifts.

`count_bits_kernighan`'s trick is worth internalizing: `n & (n - 1)` always
clears exactly the lowest set bit of `n` (subtracting 1 flips every bit
from the lowest set bit downward, and AND-ing with the original cancels
everything except that flip). It runs in a number of iterations equal to
the number of set bits, rather than the number of total bits — for a sparse
value it's meaningfully faster than shifting through every position.

### The trap: shifting by too many bits is undefined behavior

`1u << 32` on a 32-bit `unsigned int` is **undefined behavior** in C, not a
guaranteed 0 — the C standard only defines shift behavior when the shift
amount is strictly less than the operand's bit width. Compilers are free to
do anything, including constant-folding it to whatever's convenient, and
different compilers (or the same compiler at different optimization
levels) can legitimately disagree. Always know the bit width of the type
you're shifting (`sizeof(x) * 8`), and never shift by an amount that can
reach or exceed it — a common bug is shifting by a variable amount read
from user input or a file with no bounds check at all.

## A practical use: packing bytes into a 32-bit value

Bit manipulation is how binary protocols and file formats represent
multi-byte numbers as a specific sequence of bytes — this is exactly the
kind of packing [Module 08](08-networking-basics.md) needs to send integers
over a socket, and it doubles as byte-order (endianness) detection:

```c
// endian.c -- detect byte order and pack/unpack a 32-bit value into bytes
#include <stdio.h>
#include <stdint.h>

int is_little_endian(void) {
    uint32_t probe = 1;
    unsigned char *byte0 = (unsigned char *)&probe;
    return *byte0 == 1;   // low-order byte stored first == little-endian
}

void pack_be32(unsigned char out[4], uint32_t value) {
    out[0] = (value >> 24) & 0xFF;
    out[1] = (value >> 16) & 0xFF;
    out[2] = (value >> 8) & 0xFF;
    out[3] = value & 0xFF;
}

uint32_t unpack_be32(const unsigned char in[4]) {
    return ((uint32_t)in[0] << 24) | ((uint32_t)in[1] << 16) |
           ((uint32_t)in[2] << 8) | (uint32_t)in[3];
}

int main(void) {
    printf("this machine is %s-endian\n",
           is_little_endian() ? "little" : "big");

    uint32_t value = 0x11223344;
    unsigned char buf[4];
    pack_be32(buf, value);

    printf("packed big-endian bytes: %02X %02X %02X %02X\n",
           buf[0], buf[1], buf[2], buf[3]);

    uint32_t roundtrip = unpack_be32(buf);
    printf("unpacked value: 0x%08X (%s)\n",
           roundtrip, roundtrip == value ? "matches" : "MISMATCH");

    return 0;
}
```

```text
this machine is little-endian
packed big-endian bytes: 11 22 33 44
unpacked value: 0x11223344 (matches)
```

`is_little_endian` works by reading the address of a multi-byte integer as
a single `unsigned char` — on a little-endian machine (x86, Apple Silicon
running in its default mode) the *least significant byte* is stored at the
lowest address, so reading byte 0 of `1` gives `1`; on a big-endian machine
it would give `0`. `pack_be32`/`unpack_be32` sidestep the machine's native
endianness entirely by extracting and reassembling bytes explicitly with
shifts and masks — this is exactly what `htonl`/`ntohl` do in networking
code, and why network protocols specify a fixed "network byte order"
(big-endian) rather than trusting every machine to agree.

## Cheat sheet

| Task | Expression |
|---|---|
| Set bit n | `x \|= (1u << n)` |
| Clear bit n | `x &= ~(1u << n)` |
| Toggle bit n | `x ^= (1u << n)` |
| Check bit n | `(x & (1u << n)) != 0` |
| Clear lowest set bit | `x &= (x - 1)` |
| Isolate lowest set bit | `x & (-x)` (two's complement) |
| Check power of two | `x != 0 && (x & (x - 1)) == 0` |
| Count set bits | Kernighan's loop, or `__builtin_popcount(x)` on gcc/clang |

## Exercise

Write `int is_power_of_two(unsigned int x)` using the cheat-sheet idiom
above (`x != 0 && (x & (x - 1)) == 0`), and verify it against `0, 1, 2, 3,
4, 15, 16, 1024, 1023` — it should report true only for `1, 2, 4, 16, 1024`.
Then write `unsigned int reverse_bits(unsigned int x)` that reverses the
order of all 32 bits (bit 0 becomes bit 31, bit 1 becomes bit 30, and so
on) using a loop that shifts one bit out of `x` and one bit into the result
per iteration. Confirm `reverse_bits(1)` prints `2147483648` (`0x80000000`)
and `reverse_bits(0x80000000)` prints back `1`.
