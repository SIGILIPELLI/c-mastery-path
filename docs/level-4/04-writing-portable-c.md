# 04 · Writing Portable C

C runs on more platforms than any other language, and that is exactly why
portable C is hard. The standard deliberately leaves things unspecified so
that implementers can map C onto everything from an 8-bit microcontroller to
a 64-bit server, and every one of those gaps is a place your program can
behave differently somewhere else.

The useful mental model is three distinct categories, which programmers
routinely blur together:

| Category | Meaning | Example |
|---|---|---|
| **Implementation-defined** | The compiler picks and must **document** it | Is plain `char` signed? |
| **Unspecified** | The compiler picks and need not tell you | Order of evaluation of function arguments |
| **Undefined** | Anything at all may happen; no requirements | Signed overflow, out-of-bounds access |

Implementation-defined behaviour is a portability problem: your program is
still valid, it just does something else elsewhere. Undefined behaviour is a
correctness problem: your program is *invalid*, and the optimiser is
entitled to assume it cannot happen.

## What is actually guaranteed about integer types

Almost nothing. Here is what the standard promises versus what one 64-bit
machine reports:

```c
// sizes.c -- what the standard guarantees vs what this machine does
#include <stdio.h>
#include <limits.h>
#include <stdint.h>
#include <inttypes.h>
```

```text
__STDC_VERSION__ = 201112L
type         size  min   standard minimum
char            1    8   CHAR_BIT >= 8
short           2        >= 16 bits
int             4        >= 16 bits (!)
long            8        >= 32 bits
long long       8        >= 64 bits
void *          8        no guarantee at all
size_t          8        unsigned, big enough for any object

fixed width (portable):
  int32_t 4 bytes, INT32_MAX = 2147483647
  int64_t 8 bytes, INT64_MAX = 9223372036854775807
  intptr_t 8 bytes (an integer that can hold a pointer)

is plain char signed here? yes
```

`sizeof(char)` is 1 **by definition** — but a "byte" is `CHAR_BIT` bits, and
`CHAR_BIT` is only guaranteed to be at least 8. Some DSPs use 16 or 32.

The one that catches real code: `int` is only required to hold ±32767, and
`long` only ±2147483647. The three common data models disagree:

| Model | `int` | `long` | `void *` | Where |
|---|---|---|---|---|
| ILP32 | 32 | 32 | 32 | 32-bit Linux, older embedded |
| LP64 | 32 | 64 | 64 | 64-bit Linux, macOS, BSD |
| LLP64 | 32 | 32 | 64 | 64-bit Windows |

So `long` is 64-bit on macOS and 32-bit on Windows, both 64-bit platforms.
Code that stores a pointer or a file offset in a `long` is broken on
Windows and nowhere else — the single most common portability bug in C.

The fix is `<stdint.h>`: use `int32_t`/`uint64_t` when the width matters,
`size_t` for sizes and indices, `intptr_t` when an integer must hold a
pointer, and plain `int` only for small values where you genuinely do not
care.

## Format specifiers must match exactly

`printf` is variadic, so the compiler cannot coerce arguments for you — a
mismatched specifier reads the wrong number of bytes off the argument list.

```c
// badfmt.c -- format specifiers that "work" on one platform and break on another
    size_t  n   = 42;
    int64_t big = 9000000000;

    printf("%d\n", n);          /* WRONG: size_t is not int */
    printf("%ld\n", big);       /* WRONG on LLP64 (Windows): long is 32-bit there */
    printf("%zu %" PRId64 " %ld\n", n, big, l);   /* right */
```

```text
badfmt.c:11:20: warning: format specifies type 'int' but the argument has type 'size_t' (aka 'unsigned long') [-Wformat]
   11 |     printf("%d\n", n);
      |             ~~     ^
      |             %zu
badfmt.c:12:21: warning: format specifies type 'long' but the argument has type 'int64_t' (aka 'long long') [-Wformat]
      |             %lld
2 warnings generated.
```

```text
42
9000000000
42 9000000000 123
```

Both bugs printed the right answer on this machine. That is the danger:
the ABI happened to pass a 64-bit value where the format expected 32 bits,
and little-endian ordering meant the low half was in the right place.
Recompile on a big-endian or LLP64 target and you get garbage.

`%zu` for `size_t`, `%td` for `ptrdiff_t`, and the `<inttypes.h>` macros
(`PRId64`, `PRIu32`, `PRIxPTR`) are the portable answers. The macros expand
to whatever string literal is correct on the target, and adjacent string
literals concatenate — which is why the syntax looks odd.

**Treat `-Wformat` as an error.** Both bugs above were caught at compile
time by warnings that most projects have on and ignore.

## Struct layout and byte order

Never write a struct's raw bytes to a file or socket. Padding, alignment and
endianness are all implementation-defined:

```c
struct Bad  { char a; int b; char c; };   /* padded */
struct Good { int b; char a; char c; };   /* reordered */
```

```text
struct Bad : size 12 (a@0 b@4 c@8)
struct Good: size 8 (b@0 a@4 c@5)
0x01020304 in memory: 04 03 02 01 -> little-endian
portable big-endian encoding: 01 02 03 04
decoded back: 0x01020304 ok
```

The same three members, in a different order, cost 12 bytes or 8. The
compiler inserted three bytes of padding after `a` so that `b` lands on a
4-byte boundary, and one more at the end so arrays stay aligned. Ordering
members from largest to smallest usually shrinks a struct for free — a real
win when you have millions of them.

For anything that leaves the process, serialise a byte at a time:

```c
    wire[0] = (unsigned char)(v >> 24);
    wire[1] = (unsigned char)(v >> 16);
    wire[2] = (unsigned char)(v >>  8);
    wire[3] = (unsigned char)(v      );
```

This code produces big-endian output on *every* platform, because shifting
is defined in terms of values, not storage. It needs no `#ifdef`, no
endianness detection, and no `htonl`. Shift-and-mask serialisation is one of
the few places where the portable version is also the simplest.

## Implementation-defined behaviour, and where it bites

```c
// impldef.c -- undefined, unspecified, and implementation-defined, side by side
    printf("plain char is %s\n", (char)-1 < 0 ? "signed" : "unsigned");
    printf("-8 >> 1 = %d\n", -8 >> 1);
    int big = 300;
    printf("(char)300 = %d\n", (char)big);

    char c = (char)0xE9;                 /* 'é' in Latin-1 */
    printf("index from plain char : %d\n", c);
    printf("index from unsigned   : %d\n", (unsigned char)c);

    int m = INT_MAX;
    printf("INT_MAX + 1 = %d\n", m + 1);
```

```bash
clang -Wall -Wextra -std=c11 -fsanitize=undefined -o impldef impldef.c
./impldef
```

```text
plain char is signed
-8 >> 1 = -4 (arithmetic shift here; logical is also legal)
(char)300 = 44
index from plain char : -23 <-- negative! out-of-bounds index
index from unsigned   : 233
about to overflow INT_MAX...
impldef.c:24:36: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior impldef.c:24:36
INT_MAX + 1 = -2147483648
```

Four separate lessons in eight lines:

- **Plain `char` signedness is implementation-defined.** Signed on x86 and
  Apple silicon, *unsigned* on ARM Linux and PowerPC. The byte `0xE9` read
  as -23 here and would read as 233 there.
- That difference is a live bug the moment you write `table[c]`. A negative
  index is an out-of-bounds read that ASan catches and a code review does
  not. **Always use `unsigned char` for byte data**, and cast before passing
  to `<ctype.h>` functions like `isalpha`, which are defined only for
  `unsigned char` values and `EOF`.
- `-8 >> 1` giving `-4` is arithmetic shift, which every mainstream compiler
  does — but right-shifting a negative value is implementation-defined, so
  it is not guaranteed. Shift unsigned types; divide signed ones.
- Signed overflow is **undefined**, not "wraps around." UBSan named the line.
  It printed `-2147483648` this time; at a different optimisation level the
  compiler may delete the check that guards it, because it is allowed to
  assume overflow never happens. Use `unsigned` arithmetic when you want
  defined wrapping, or check before the operation
  (`if (a > INT_MAX - b) ...`).

## Feature test macros and platform detection

Anything beyond ISO C — POSIX calls, GNU extensions — needs the right
feature macro, **before any include**:

```c
#define _POSIX_C_SOURCE 200809L   /* must precede every #include */
#include <unistd.h>
```

For genuinely divergent platforms, detect at compile time and keep the
`#ifdef` blocks as small as possible — ideally one wrapper function, not
sprinkled through your logic:

```c
#if defined(_WIN32)
    #define PATH_SEP '\\'
#elif defined(__APPLE__)
    #define PATH_SEP '/'
#elif defined(__linux__)
    #define PATH_SEP '/'
#else
    #error "unsupported platform"
#endif
```

The `#error` is deliberate. A silent fallback that "probably works" on an
unknown platform is worse than a build failure that tells someone to think.

## Cheat sheet

| Need | Portable answer |
|---|---|
| Exact-width integer | `int32_t`, `uint64_t` (`<stdint.h>`) |
| Size of an object / an index | `size_t` |
| Difference of two pointers | `ptrdiff_t` |
| Integer holding a pointer | `intptr_t` / `uintptr_t` |
| Print them | `%zu`, `%td`, `PRId64`, `PRIxPTR` (`<inttypes.h>`) |
| Byte data | `unsigned char`, never plain `char` |
| Type limits | `<limits.h>`, `<float.h>` |
| Alignment of a type | `_Alignof(T)` (C11) |
| Offset of a member | `offsetof(T, m)` (`<stddef.h>`) |
| Serialise a number | Shift and mask into `unsigned char[]` |
| Detect C version | `__STDC_VERSION__` (`201112L` = C11, `201710L` = C17) |
| Detect platform | `_WIN32`, `__linux__`, `__APPLE__`, `__unix__` |
| Enable POSIX APIs | `#define _POSIX_C_SOURCE 200809L` first |

Portability flags worth turning on: `-std=c11 -pedantic -Wall -Wextra`
rejects extensions you did not mean to use, and
`-Wconversion -Wsign-conversion` finds the implicit narrowing that silently
changes behaviour when a type's width changes underneath you.

## Exercise

Write `portable_io.c` providing `write_u32(FILE *, uint32_t)` and
`read_u32(FILE *, uint32_t *)` that use shift-and-mask serialisation, plus a
`write_struct`/`read_struct` pair for a record containing a `uint16_t`, a
`uint64_t`, and a fixed 12-byte name. Write a file with one program and read
it back with another; verify the on-disk size is exactly what you computed
by hand, not `sizeof(struct)`.

Then hunt for portability bugs in your own earlier code. Rebuild any program
from Level 3 with `-std=c11 -pedantic -Wall -Wextra -Wconversion` and fix
every warning. Pay particular attention to: `int` used where `size_t`
belongs, `%d` printing a `size_t`, plain `char` holding data above 127, and
any `sizeof(long)` assumption. For each fix, write one sentence naming the
platform where the original would actually have broken — if you cannot name
one, you may not have found a real bug.
