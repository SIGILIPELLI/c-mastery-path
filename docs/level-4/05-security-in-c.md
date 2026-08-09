# 05 · Security in C

Most C security bugs are not exotic. They are ordinary mistakes — a copy
without a bound, a size computed by multiplication, a user string passed
where a format string was expected — that happen to sit on a path an
attacker controls. The language will not stop any of them, because C's
design contract is that the programmer knows what they are doing.

The most dangerous property of these bugs is that **they usually do not
crash**. A crash is the good outcome; it means someone notices. The first
example is a program that overflows a buffer, exits with status 0, prints no
warning, and grants administrator access.

## Buffer overflow: the one that does not crash

```c
// escalate.c -- an overflow that does not crash: it changes a decision
#include <stdio.h>
#include <string.h>

struct Login { char user[16]; int is_admin; };

/* Copying a caller-controlled length into a fixed buffer. */
static void copy_name(char *dst, const char *src, size_t n) {
    memcpy(dst, src, n);                /* n comes from the attacker */
}

int main(int argc, char **argv) {
    struct Login l;
    memset(&l, 0, sizeof l);

    const char *input = argc > 1 ? argv[1] : "alice";
    copy_name(l.user, input, strlen(input) + 1);

    printf("user=%-22s is_admin=%d  -> %s\n", l.user, l.is_admin,
           l.is_admin ? "ADMIN ACCESS GRANTED" : "normal user");
    return 0;
}
```

```bash
clang -Wall -Wextra -fno-stack-protector -o escalate escalate.c
./escalate alice
./escalate "AAAAAAAAAAAAAAAA1"
```

```text
user=alice                  is_admin=0  -> normal user
user=AAAAAAAAAAAAAAAA1      is_admin=49  -> ADMIN ACCESS GRANTED
exit=0
```

Sixteen `A`s fill `user` exactly. The seventeenth character, `1`, lands in
the first byte of `is_admin` — which becomes 49, the ASCII code for `'1'`.
Nonzero is true, and the program grants admin.

No crash. No warning. Exit status 0. This is what memory corruption
actually looks like in the field: not a segfault, but a program that quietly
does the wrong thing. The attacker did not need shellcode, a debugger, or
knowledge of the stack layout — just one character too many.

Compile the same code with the sanitizer and the invisible becomes obvious:

```bash
clang -Wall -Wextra -fsanitize=address -g -o escalate_asan escalate.c
./escalate_asan "AAAAAAAAAAAAAAAA1"
```

```text
==22632==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x00016bdc29b4
WRITE of size 25 at 0x00016bdc29b4 thread T0
    #1 0x00010403c9b4 in authenticate overflow.c:10
    #2 0x00010403c86c in main overflow.c:15

Address 0x00016bdc29b4 is located in stack of thread T0 at offset 52 in frame
```

Exact line, exact byte, exact frame. **Run your test suite under
`-fsanitize=address` at least once.** It is the single highest-value thing
in this module.

Modern compilers also add defences by default. The same overflow written
with a literal `strcpy` into a fixed array is caught at runtime by the
compiler's own bounds check (`_FORTIFY_SOURCE`), which aborts with `SIGTRAP`
— exit status 133. Those defences are real but partial: `escalate.c` slips
past them because the length is a runtime value that the compiler cannot
reason about. Never treat a hardening flag as a substitute for a bound.

## Safe string handling: `strncpy` is not the safe one

The standard "safe" replacements have their own traps.

```c
// strings.c -- three "safe" copies, only two of which are
    const char *src = "a string longer than the buffer";

    char a[16];
    strncpy(a, src, sizeof a);          /* NO null terminator when it truncates */

    char b[16];
    strncpy(b, src, sizeof b - 1);
    b[sizeof b - 1] = '\0';             /* the fix strncpy requires */

    char c[16];
    int need = snprintf(c, sizeof c, "%s", src);
```

```text
strncpy  : a string longer    <- last byte is ' ', not '\0'
strncpy+ : "a string longer" (len 15)
snprintf : "a string longer" (len 15)
           returned 31 = length it WANTED; truncated, detectable
```

`strncpy` copied exactly 16 bytes and stopped — leaving **no null
terminator**. Every subsequent `strlen`, `printf("%s")` or `strcat` on that
buffer reads past the end until it happens to find a zero byte. `strncpy`
was designed in the 1970s for fixed-width record fields, not for safe string
copying, and it is a frequent source of the very bug people use it to avoid.

`snprintf` is the one to reach for. It always null-terminates, and its
return value is **the length it would have needed** — so
`if (n >= sizeof buf)` detects truncation. Silent truncation is its own
vulnerability class: a path checked as `/safe/dir/file` and then truncated
to `/safe/dir` is a different path than the one you validated.

## Format string vulnerabilities

```c
// fmtstr.c -- user input used as a format string
    printf(input);                  /* attacker controls the format */
    printf("%s\n", input);          /* input is data, not a format */
```

```text
fmtstr.c:7:12: warning: format string is not a string literal (potentially insecure) [-Wformat-security]
    7 |     printf(input);
      |            ^~~~~
      |            "%s",
```

```text
--- benign ---
unsafe: hello
safe:   hello
--- attacker input ---
unsafe: 1 f33efa60 6d7b7379 0x16d7b7379
safe:   %x %x %x %p
```

Passing `%x %x %x %p` made `printf` walk the argument registers and stack
that no caller ever populated, dumping raw process memory — enough to defeat
ASLR and stack-canary protections by leaking the addresses they rely on.
`%n` goes further and *writes* to memory. The rule is absolute: **a format
string must be a literal.** If it must be dynamic, it must come from a
fixed set you control, never from input.

Turn this one into an error: `-Wformat-security -Werror=format-security`.

## Integer overflow in allocation sizes

`malloc(count * size)` is a vulnerability whenever `count` comes from
outside:

```c
// intovf.c -- an allocation size that wraps
static void *alloc_array_bad(size_t count, size_t size) {
    return malloc(count * size);            /* can wrap to a tiny value */
}

static void *alloc_array_good(size_t count, size_t size) {
    if (size != 0 && count > SIZE_MAX / size) {
        fprintf(stderr, "  refused: %zu * %zu would overflow\n", count, size);
        return NULL;                        /* check BEFORE multiplying */
    }
    return malloc(count * size);
}
```

```text
count = 4611686018427387905, size = 4
count * size wraps to 4 bytes
alloc_array_bad  -> non-NULL  <-- a tiny buffer for a huge array
alloc_array_good -> NULL

calloc does the check for you:
calloc(4611686018427387905, 4) -> NULL (correctly refused)
```

The multiplication wrapped to **4**. `malloc` cheerfully returned a 4-byte
buffer, and the code that follows will write four billion elements into it —
a heap overflow with a completely attacker-chosen size.

The check must happen **before** the multiplication, because after it the
evidence is gone: `count > SIZE_MAX / size` uses division, which cannot
overflow. `size_t` is unsigned so this wrap is well-defined rather than UB,
which makes it more dangerous, not less — no sanitizer flags it by default.
`calloc` performs this check for you and is the better default for arrays.

## Cheat sheet

| Never use | Use instead | Why |
|---|---|---|
| `gets()` | `fgets()` | No bound at all; removed from C11 |
| `strcpy` / `strcat` | `snprintf` | No bound |
| `sprintf` | `snprintf` | No bound |
| `strncpy` | `snprintf` | May not null-terminate |
| `printf(user)` | `printf("%s", user)` | Format string attack |
| `atoi` | `strtol` + `errno` | No error or overflow reporting |
| `malloc(n * sz)` | `calloc(n, sz)` or check first | Multiplication overflow |
| `scanf("%s", buf)` | `scanf("%15s", buf)` / `fgets` | Unbounded write |

| Flag | Effect |
|---|---|
| `-fsanitize=address` | Catches overflow, use-after-free, double free |
| `-fsanitize=undefined` | Catches signed overflow, bad shifts, misaligned access |
| `-D_FORTIFY_SOURCE=2 -O2` | Compile-time-known bounds checked at runtime |
| `-fstack-protector-strong` | Detects stack smashing via a canary |
| `-Wformat-security -Werror=format-security` | Rejects non-literal format strings |
| `-Wall -Wextra -Wconversion` | Catches the narrowing that precedes overflow |
| `-fPIE -pie`, `-Wl,-z,relro,-z,now` | ASLR and hardened relocations (Linux) |

The other three bug classes to internalise:

- **Use-after-free / double free.** A pointer is not invalidated by `free`.
  Set it to `NULL` immediately after freeing; `free(NULL)` is a guaranteed
  no-op, so a double free becomes harmless.
- **Off-by-one.** `for (i = 0; i <= n; i++)` on an `n`-element array writes
  one past the end — often exactly the byte holding a length, a flag, or a
  saved pointer, as `escalate.c` showed.
- **TOCTOU.** Checking a file with `access()` and then `open()`ing it lets an
  attacker swap the file in between. Open first, then check the *file
  descriptor* with `fstat`.

Finally: **all input is hostile**, including input from files you wrote,
environment variables, `argv`, and other processes on the same machine.
Validate length before copying, range before indexing, and sign before
converting to `size_t` — a negative `int` becomes an enormous unsigned value
and turns a length check into a no-op.

## Exercise

Take the `tiny_http.c` server from
[Level 3 module 08](../level-3/08-networking-basics.md) and audit it as an
attacker. Find and fix every issue: the single unbounded `recv`, the
`sscanf` widths, any path that could be used to read outside an intended
directory, and what happens when a client sends 4 KB with no spaces or
newlines at all. Then rebuild it with
`-fsanitize=address,undefined -D_FORTIFY_SOURCE=2 -O2` and fire malformed
requests at it with `printf` piped into `nc` — empty requests, a request
line of 10,000 bytes, a request containing `%n%n%n`, and one with embedded
null bytes.

Then write the fix you will reuse forever: a `safe_copy(char *dst, size_t
dstsz, const char *src)` that returns the number of bytes it *wanted* to
write (so callers can detect truncation), always null-terminates, and
handles `dstsz == 0` without writing anything at all. Prove all three
properties with tests, then check the last one under
`-fsanitize=address` — the zero-size case is where nearly every hand-rolled
version has its bug.
