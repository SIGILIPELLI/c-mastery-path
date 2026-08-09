# 08 · Testing in C

C has no built-in test framework, no assertion library, and no test runner.
What it has instead is enough preprocessor machinery to build all three in
about forty lines — and, more importantly, a set of runtime tools
(sanitizers, coverage instrumentation) that catch a category of bug no
assertion ever will.

The plan: build a minimal framework, use it on a small string library, then
add the three things that turn a passing test suite into a *trustworthy*
one — sanitizers, coverage, and property tests.

## A unit test framework in forty lines

```c
/* munit.h -- a unit test framework in 40 lines of macros */
#ifndef MUNIT_H
#define MUNIT_H

#include <stdio.h>
#include <string.h>

static int mu_tests = 0, mu_failures = 0, mu_current_failed = 0;

#define MU_FAIL(fmt, ...) do {                                            \
    printf("  FAIL %s:%d: " fmt "\n", __FILE__, __LINE__, __VA_ARGS__);   \
    mu_current_failed = 1;                                                \
} while (0)

#define ASSERT_EQ_INT(expected, actual) do {                              \
    long _e = (long)(expected), _a = (long)(actual);                      \
    if (_e != _a) MU_FAIL("%s: expected %ld, got %ld", #actual, _e, _a);  \
} while (0)

#define ASSERT_EQ_STR(expected, actual) do {                              \
    const char *_e = (expected), *_a = (actual);                          \
    if (_a == NULL || strcmp(_e, _a) != 0)                                \
        MU_FAIL("%s: expected \"%s\", got \"%s\"", #actual, _e,           \
                _a ? _a : "(null)");                                      \
} while (0)

#define RUN_TEST(fn) do {                                                 \
    mu_tests++; mu_current_failed = 0;                                    \
    printf("%-28s", #fn);                                                 \
    fn();                                                                 \
    if (mu_current_failed) { mu_failures++; }                             \
    else printf(" ok\n");                                                 \
} while (0)

#define MU_REPORT() (                                                     \
    printf("\n%d tests, %d failed\n", mu_tests, mu_failures),             \
    mu_failures == 0 ? 0 : 1)

#endif
```

Four preprocessor techniques are doing all the work here:

- **`do { ... } while (0)`** wraps every multi-statement macro. Without it,
  `if (x) ASSERT_EQ_INT(1, y); else ...` breaks, because the expansion's
  trailing `;` terminates the `if`. This is the standard C idiom for a
  macro that must behave like one statement.
- **`#expr` stringifies** its argument, so the failure message contains the
  literal source text of the expression that failed.
- **`__FILE__` and `__LINE__`** expand at the *use* site, not in the header,
  so you get the line number of the failing assertion.
- **`_e` and `_a` locals** evaluate each argument exactly once.
  `ASSERT_EQ_INT(0, pop(stack))` must not call `pop` twice, and a naive
  macro would.

`MU_REPORT()` returns 1 on failure, which becomes the process exit status —
the only thing `make`, CI, and every test runner actually look at.

## The code under test

```c
// strutil.c
/* Trim leading and trailing whitespace IN PLACE. Returns the new start. */
char *str_trim(char *s) {
    while (*s && isspace((unsigned char)*s)) s++;
    if (*s == '\0') return s;
    char *end = s + strlen(s) - 1;
    while (end > s && isspace((unsigned char)*end)) end--;
    end[1] = '\0';
    return s;
}

/* Copy with truncation. Returns the length it WANTED to write. */
size_t str_copy(char *dst, size_t dstsz, const char *src) {
    size_t len = strlen(src);
    if (dstsz == 0) return len;
    size_t n = (len < dstsz - 1) ? len : dstsz - 1;
    memcpy(dst, src, n);
    dst[n] = '\0';
    return len;
}
```

(The `(unsigned char)` casts before `isspace` are the portability issue from
[module 04](04-writing-portable-c.md), not decoration.)

## Tests that target boundaries, not the happy path

```c
static void test_copy_truncates(void) {
    char dst[6];
    ASSERT_EQ_INT(11, str_copy(dst, sizeof dst, "hello world"));  /* wanted 11 */
    ASSERT_EQ_STR("hello", dst);                                  /* got 5 + NUL */
}

static void test_copy_zero_size(void) {
    char dst[1] = { 'X' };
    ASSERT_EQ_INT(5, str_copy(dst, 0, "hello"));
    ASSERT_EQ_INT('X', dst[0]);          /* must not write anything */
}

static void test_trim_all_whitespace(void) {
    char buf[] = "    ";
    ASSERT_EQ_STR("", str_trim(buf));
}
```

```bash
clang -Wall -Wextra -std=c11 -fsanitize=address,undefined -g \
      -o test_strutil test_strutil.c strutil.c
./test_strutil
```

```text
test_trim_both_sides         ok
test_trim_all_whitespace     ok
test_trim_empty              ok
test_copy_fits               ok
test_copy_truncates          ok
test_copy_zero_size          ok
test_count_char              ok

7 tests, 0 failed
```

Notice which cases got tests: empty string, all-whitespace string, exact
truncation, and **zero destination size**. In C the interesting inputs are
almost always the boundaries — 0, 1, exactly-n, n+1, NULL — because that is
where the off-by-one lives.

`char buf[] = "   hello   "` rather than `char *buf = "..."` is deliberate
too. `str_trim` writes a null byte, and writing to a string literal is
undefined behaviour that typically segfaults. Array initialisation makes a
modifiable copy.

## Always build tests with sanitizers

This is the highest-value line in the module. Here is the suite run against
a version of `str_copy` missing exactly one line — the `if (dstsz == 0)`
guard:

```text
==23775==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x00016ae768a1
WRITE of size 5 at 0x00016ae768a1 thread T0
    #1 0x000104f8a890 in str_copy strutil_buggy.c:20
    #2 0x000104f89e2c in test_copy_zero_size test_strutil.c:33
    #3 0x000104f88d64 in main test_strutil.c:49

Address 0x00016ae768a1 is located in stack of thread T0 at offset 33 in frame
    #0 0x000104f89cf8 in test_copy_zero_size test_strutil.c:31

  This frame has 1 object(s):
    [32, 33) 'dst' (line 32) <== Memory access at offset 33 overflows this variable
SUMMARY: AddressSanitizer: stack-buffer-overflow strutil_buggy.c:20 in str_copy
```

The buggy line, the test that reached it, and the exact variable overflowed.
Without ASan this test might well have *passed*: `dstsz - 1` with `dstsz == 0`
wraps to `SIZE_MAX`, `n` becomes `strlen(src)`, and the copy scribbles five
bytes past a one-byte buffer into stack padding that nothing else was using
yet. The assertion on `dst[0]` would catch it here — but move that variable
and it would not.

A C test suite that does not run under `-fsanitize=address,undefined` is
testing your logic and ignoring your memory safety, which is the half of C
that actually breaks.

## Coverage tells you what you did not test

```bash
clang -Wall -Wextra -std=c11 -fprofile-instr-generate -fcoverage-mapping -g \
      -o test_cov test_strutil.c strutil.c
LLVM_PROFILE_FILE=t.profraw ./test_cov
llvm-profdata merge -sparse t.profraw -o t.profdata
llvm-cov report ./test_cov -instr-profile=t.profdata strutil.c
```

```text
Filename      Regions  Missed  Cover   Functions  Missed  Executed  Lines  Missed  Cover   Branches  Missed  Cover
strutil.c          28       0  100.00%         3       0   100.00%     20       0  100.00%       18       1  94.44%
```

100% of lines, but **94.44% of branches** — one branch never taken in either
direction. That gap is the useful number. Line coverage is easy to max out
and says little; branch coverage finds the `else` you never exercised and
the loop that never ran zero times.

Use coverage to find untested code, never as a target. 100% line coverage
with no assertions proves only that the code did not crash.

## Property tests: invariants over generated inputs

Hand-written cases test the inputs you thought of. Property tests state a
rule that must hold for *all* inputs, then throw generated data at it:

```c
// proptest.c -- assert invariants over random inputs, not hand-picked ones
        random_string(src, 60, &seed);
        strcpy(work, src);
        char *t = str_trim(work);

        /* Property 1: the result never starts or ends with whitespace. */
        if (*t) {
            if (isspace((unsigned char)t[0]))             { puts("P1a"); return 1; }
            if (isspace((unsigned char)t[strlen(t) - 1])) { puts("P1b"); return 1; }
        }

        /* Property 2: trimming is idempotent. */
        char again[64]; strcpy(again, t);
        if (strcmp(str_trim(again), t) != 0)              { puts("P2"); return 1; }

        /* Property 3: str_copy always NUL-terminates and never overflows. */
        size_t want = str_copy(dst, sizeof dst, t);
        if (want != strlen(t))                            { puts("P3a"); return 1; }
        if (strlen(dst) != (want < sizeof dst ? want : sizeof dst - 1))
                                                          { puts("P3b"); return 1; }
```

```text
200000 random inputs, all properties held
  trimmed to empty : 11746
  truncated copies : 156480
```

Two hundred thousand inputs, run under ASan and UBSan, exercising 11,746
all-whitespace strings and 156,480 truncating copies — far more edge cases
than anyone writes by hand. The alphabet is deliberately whitespace-heavy
(`"ab \t\n  "`) so that interesting cases actually occur; uniformly random
bytes would almost never produce an all-whitespace string.

The counters at the end matter as much as the pass. A property test that
reports zero truncations is testing nothing about truncation, however green
it looks.

For serious work, graduate to a **coverage-guided fuzzer**. Clang's
libFuzzer (`-fsanitize=fuzzer,address`) needs only a
`LLVMFuzzerTestOneInput(const uint8_t *data, size_t size)` entry point and
will then mutate inputs to maximise new code paths — dramatically better
than random generation at reaching deep branches. AFL++ does the same for
programs that read stdin or a file.

## Cheat sheet

| Technique | Catches |
|---|---|
| Unit tests on boundaries (0, 1, n, n+1, NULL) | Off-by-one, empty-input handling |
| `-fsanitize=address` | Overflow, use-after-free, double free, leaks |
| `-fsanitize=undefined` | Signed overflow, bad shift, misaligned access |
| `-fsanitize=thread` | Data races that did not manifest |
| `llvm-cov` / `gcov` branch coverage | Code you never exercised |
| Property tests | Cases you never imagined |
| libFuzzer / AFL++ | Deep paths reachable only by odd input |
| `assert()` in the code itself | Broken internal invariants, in debug builds |

| Framework | Notes |
|---|---|
| Hand-rolled macros | Zero dependencies; what this module built |
| Unity | Tiny, embedded-friendly, C89 |
| Check | Forks per test, so a crash does not kill the run |
| Criterion | Auto-registers tests, parameterised, modern |
| CMocka | Mocking and fixtures |

Testing traps specific to C:

- **`assert()` vanishes under `-DNDEBUG`.** Never put an operation with side
  effects inside one — `assert(pop(&stack) == 3)` stops popping in release
  builds.
- **A test that crashes takes the runner with it.** Frameworks like Check
  fork per test for exactly this reason; hand-rolled runners lose every
  result after the crash.
- **Global state persists between tests.** A file-scope table left dirty by
  test 3 makes test 7 pass for the wrong reason. Reset explicitly in setup,
  and run tests in a different order sometimes to check.
- **Testing only the `-O0` build.** Undefined behaviour frequently changes
  shape with optimisation, as [module 02](02-concurrency-synchronization.md)
  showed. Run the suite at `-O0` and `-O2`.

## Exercise

Extend `munit.h` with the two features it most obviously lacks: a
`ASSERT_EQ_MEM(expected, actual, n)` for comparing byte buffers that prints
the first differing offset and both bytes in hex, and a fixture mechanism —
`RUN_TEST_F(setup, fn, teardown)` — that runs setup before and teardown
after, *even when the test fails*.

Then apply the whole toolkit to the AVL tree from
[module 06](06-advanced-data-structures.md). Write unit tests for the
boundaries (empty tree, single node, duplicate key, all four rotation
cases), measure branch coverage and add tests until every rotation branch is
hit, then write a property test that inserts random keys and asserts the
balance invariant and in-order sortedness after every single insert. Run all
of it under `-fsanitize=address,undefined` at both `-O0` and `-O2`.

Finally, do it in the honest order: write the tests for `avl_delete`
*before* implementing it, watch them fail with useful messages, and only
then write the code. If a failure message does not tell you what went wrong
without opening a debugger, improve the assertion macro rather than the
test.
