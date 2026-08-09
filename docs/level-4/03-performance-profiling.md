# 03 · Performance Optimization & Profiling

The rule that governs this entire module: **measure, fix the biggest thing,
measure again.** Programmers are famously bad at guessing where time goes,
and C makes it worse — the code you wrote and the code that runs are
different, sometimes dramatically.

The worked example here is a word-frequency counter that takes half a
second to do 60,000 lookups. We will find out why with a real profiler,
fix it, and get an 83x speedup — then compare that against what the compiler
alone can do for you.

## Step one: time it, honestly

Before any profiler, you need a number you trust. `clock_gettime` with
`CLOCK_MONOTONIC` is the right tool: it measures elapsed wall time and never
jumps backwards when the system clock is adjusted.

```c
static double now(void) {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec + ts.tv_nsec / 1e9;
}
```

Avoid `clock()`, which measures CPU time in implementation-defined units and
counts *all* threads, so a parallel program appears to get slower as it gets
faster. Time only the region under study, run it more than once, and ignore
the first run — it pays for cold caches and page faults.

Here is the subject:

```c
// wordfreq.c -- deliberately naive: a linear scan where a hash table belongs
#define MAXW  4000
#define WORDS 60000

typedef struct { char word[16]; int count; } Slot;

static Slot table[MAXW];
static int nslots = 0;

/* O(n) scan of every known word, for every input word. */
static int find_slot(const char *w) {
    for (int i = 0; i < nslots; i++)
        if (strcmp(table[i].word, w) == 0) return i;
    return -1;
}

static void tally(const char *w) {
    int i = find_slot(w);
    if (i >= 0) { table[i].count++; return; }
    if (nslots < MAXW) {
        snprintf(table[nslots].word, sizeof table[nslots].word, "%s", w);
        table[nslots].count = 1;
        nslots++;
    }
}
```

```bash
clang -Wall -Wextra -std=c11 -O2 -o wordfreq wordfreq.c
./wordfreq
```

```text
distinct words : 4000
total counted  : 60000
elapsed        : 0.504 s
```

Half a second for 60,000 lookups is absurd, but "absurd" is a feeling. The
next step turns it into a count.

## Step two: instrumented profiling with llvm-cov

Clang can compile counters into every branch and every line. Combined with
`llvm-profdata`, this gives **exact** execution counts — not samples, not
estimates:

```bash
clang -Wall -Wextra -std=c11 -O2 -fprofile-instr-generate -fcoverage-mapping \
      -o wordfreq_prof wordfreq.c
LLVM_PROFILE_FILE=wf.profraw ./wordfreq_prof
llvm-profdata merge -sparse wf.profraw -o wf.profdata
llvm-cov show ./wordfreq_prof -instr-profile=wf.profdata -name=find_slot
```

```text
wordfreq.c:find_slot:
   21|  60.0k|static int find_slot(const char *w) {
   22|   120M|    for (int i = 0; i < nslots; i++)
   23|   120M|        if (strcmp(table[i].word, w) == 0) return i;
   24|  4.00k|    return -1;
   25|  60.0k|}
```

There is the answer, in one screen. 60,000 calls to `find_slot` executed
**120 million** `strcmp` calls. The average lookup compares against 2,000
words, because the table fills to 4,000 entries and the scan is linear.

The arithmetic checks out and confirms the reading: 60,000 lookups × ~2,000
average comparisons ≈ 120 million. That is not a constant-factor problem, it
is an algorithmic one, and no compiler flag will rescue it.

## Step three: sampling, for when you cannot rebuild

Instrumentation requires a special build. A **sampling** profiler
interrupts a running process and records the stack, so it works on release
binaries and running servers. On macOS that is `sample`; on Linux,
`perf record` / `perf report`.

```bash
./wordfreq &
sample $(pgrep -n wordfreq) 1
```

```text
Call graph:
    311 Thread_467866   DispatchQueue_1: com.apple.main-thread  (serial)
      311 start  (in dyld) + 6992
        311 main  (in wf_O0) + 140
          311 tally  (in wf_O0) + 24
            226 find_slot  (in wf_O0) + 80
            + 169 _platform_strcmp  (in libsystem_platform.dylib)
            + 29 DYLD-STUB$$strcmp  (in wf_O0)
             85 find_slot  (in wf_O0) + 32,72,...
```

Every one of the 311 samples landed inside `tally`, and 226 of them inside
`find_slot` — with 169 in `strcmp` itself. The two techniques agree, which
is the point: instrumentation tells you *how many times*, sampling tells you
*where the time is*, and a real investigation uses both.

Read a sampling profile carefully. The numbers are inclusive down the call
graph, so `main` having 311 samples means "everything happened under main,"
not "main is slow." What you want is the deepest frame with a large
**self** count — here, `strcmp`.

## Step four: fix the algorithm

Replace the linear scan with an open-addressed hash index. Same output, same
structure, one different lookup:

```c
// wordfreq2.c -- same program, hash lookup instead of a linear scan
#define BUCKETS 8192            /* power of two, >2x the distinct words */

static int index_of[BUCKETS];   /* bucket -> slot index, -1 for empty */

static unsigned hash(const char *s) {
    unsigned h = 2166136261u;
    for (const unsigned char *p = (const unsigned char *)s; *p; p++)
        h = (h ^ *p) * 16777619u;
    return h;
}

static void tally(const char *w) {
    unsigned b = hash(w) & (BUCKETS - 1);        /* mask, not % */
    while (index_of[b] != -1) {                  /* open addressing */
        if (strcmp(table[index_of[b]].word, w) == 0) {
            table[index_of[b]].count++;
            return;
        }
        b = (b + 1) & (BUCKETS - 1);
    }
    if (nslots < MAXW) {
        snprintf(table[nslots].word, sizeof table[nslots].word, "%s", w);
        table[nslots].count = 1;
        index_of[b] = nslots++;
    }
}
```

```text
wordfreq2:  distinct words : 4000  total counted : 60000  elapsed : 0.006 s
wordfreq :  distinct words : 4000  total counted : 60000  elapsed : 0.497 s
```

0.497 s → 0.006 s. **83x**, from changing the lookup and nothing else.

Two small things in that code are worth noticing. `& (BUCKETS - 1)` replaces
`% BUCKETS` — legal only because `BUCKETS` is a power of two, and worth it
because integer division is one of the slowest instructions on most CPUs.
And the table is sized at more than twice the expected entries: open
addressing degrades sharply above about 70% load factor, at which point you
have reinvented the linear scan.

## Step five: check what the compiler can do (spoiler: less)

Same naive program, four optimisation levels:

```text
-O0 : 0.624 s
-O1 : 0.509 s
-O2 : 0.509 s
-O3 : 0.510 s
```

From `-O0` to `-O3`: **1.2x**. From the algorithm change: **83x**. `-O3` did
not help at all over `-O1` here, because there is nothing to vectorise or
unroll — the program is spending its life in `strcmp` on data it should
never have been looking at.

This is the shape of most real performance work. Compiler flags are worth
maybe 2–3x on typical code; algorithms and memory layout are worth orders of
magnitude. Turn on `-O2`, then go find your `find_slot`.

## Memory layout is the other order of magnitude

The one case where "the same algorithm" genuinely differs by 10x is memory
access order. Identical arithmetic, identical instruction count, one array
traversed two ways:

```c
// cache.c -- identical arithmetic, two traversal orders
    for (int i = 0; i < N; i++)
        for (int j = 0; j < N; j++)
            rows += m[i][j];              /* memory order: stride 1 */

    for (int j = 0; j < N; j++)
        for (int i = 0; i < N; i++)
            cols += m[i][j];              /* stride N: a new cache line each time */
```

```text
sums equal: yes (16777216)
row-major traversal    : 0.008 s
column-major traversal : 0.094 s
penalty                : 11.1x
```

C stores 2-D arrays **row-major**: `m[i][j]` and `m[i][j+1]` are adjacent in
memory. The first loop walks straight through, and each 64- or 128-byte
cache line fetched serves the next dozen-plus iterations. The second jumps
`N * sizeof(int)` bytes every step, so nearly every read is a cache miss and
the hardware prefetcher cannot predict it.

The fix is almost always to reorder the loops, not to add threads. An
11x-slower loop run on eight cores is still slower than the fast loop on
one.

## Cheat sheet

| Tool | Kind | What it gives you |
|---|---|---|
| `clock_gettime(CLOCK_MONOTONIC)` | Manual | Trustworthy wall time for a region |
| `-fprofile-instr-generate -fcoverage-mapping` + `llvm-cov show` | Instrumented | Exact per-line execution counts |
| `gprof` (with `-pg`) | Instrumented | Call counts and flat profile; Linux/GCC |
| `perf record` / `perf report` | Sampling | Where time goes in a release build (Linux) |
| `sample <pid>` | Sampling | Same, on macOS; also `xctrace` for Instruments |
| `perf stat` | Counters | Cache misses, branch mispredicts, IPC |
| `-O2` | Compiler | The sane default; `-O3` rarely adds much |
| `-march=native` | Compiler | Uses this CPU's instructions — breaks portability |
| `-flto` | Compiler | Cross-file inlining at link time |
| `-fno-omit-frame-pointer` | Compiler | Keeps profiler stacks readable |

Traps that specifically bite C programmers here:

- **Timing a `-O0` build.** Debug builds can be 5–10x slower and have
  completely different hotspots. Profile what you ship.
- **Benchmarking code the optimiser deleted.** If a result is unused, the
  loop computing it can vanish entirely — you measure nothing and conclude
  it is infinitely fast. Always consume the result (print it, accumulate it
  into a `volatile`), as every example here does.
- **Measuring once.** Run each variant several times. The 83x above held
  across runs; a 5% difference across single runs is noise.
- **Optimising before profiling.** The `strcmp` in `find_slot` was never the
  problem. Replacing it with a faster comparison would have won a few
  percent of a fundamentally wrong design.

## Exercise

Profile and fix a second hotspot in `wordfreq2.c`. After the hash change,
build it with `-fprofile-instr-generate -fcoverage-mapping` and find where
the remaining 0.006 s goes — you should discover that `snprintf` in the
input loop now dominates, because formatting a string is far more expensive
than looking one up. Replace it with manual digit formatting and measure
what you gain.

Then quantify the cache effect properly. Modify `cache.c` to take the stride
as a command-line argument (traversing `m[0][0]`, `m[0][s]`, `m[0][2s]`, …)
and sweep `s` from 1 to 64. Plot time against stride and identify the point
where performance flattens out — that is where every access already misses,
so a larger stride costs nothing more. Explain why the curve is a step
rather than a straight line, and relate the step position to your cache line
size divided by `sizeof(int)`.
