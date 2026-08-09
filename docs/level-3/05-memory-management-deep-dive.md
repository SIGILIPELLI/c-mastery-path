# 05 · Memory Management Deep Dive

[Level 2](../level-2/02-dynamic-memory.md) introduced `malloc` and `free`.
This module goes one level deeper: *where* in memory your variables
actually live, what `realloc` really does under the hood, and how to
systematically catch the bugs — leaks, use-after-free, double-free — that
don't always crash the moment they happen. These are the exact bugs that
show up hardest to debug in [Module 10](10-project-kv-store.md)'s
multi-threaded server, where a bad free can corrupt state used by a
completely different thread.

## Four regions, one address space

A running C program's memory is divided into regions with different
lifetimes:

```c
// layout.c -- where different kinds of variables actually live
#include <stdio.h>
#include <stdlib.h>

int global_var = 100;             // data segment (initialized global)
static int static_var;            // BSS segment (zero-initialized)

void show_addresses(void) {
    int local_var = 1;            // stack
    int *heap_var = malloc(sizeof(int));
    *heap_var = 1;

    printf("global_var (data):   %p\n", (void *)&global_var);
    printf("static_var (bss):    %p\n", (void *)&static_var);
    printf("local_var (stack):   %p\n", (void *)&local_var);
    printf("heap_var  (heap):    %p\n", (void *)heap_var);

    free(heap_var);
}

int main(void) {
    show_addresses();
    return 0;
}
```

```bash
gcc -Wall -Wextra -g -O0 -o layout layout.c
./layout
```

```text
global_var (data):   0x1007dc000
static_var (bss):    0x1007dc004
local_var (stack):   0x16f62aa5c
heap_var  (heap):    0x100e9d940
```

(The exact addresses will differ on every run — modern OSes randomize
layout per-process for security. What stays constant is the *grouping*:
`global_var` and `static_var` sit next to each other in the low, static
part of the address space; `local_var` lives on the stack, which grows and
shrinks automatically with function calls; `heap_var` lives in a
completely separate region that only `malloc`/`free` manage.)

| Region | Lifetime | Managed by |
|---|---|---|
| Data / BSS | Program start to program exit | Compiler/linker (fixed size, decided at compile time) |
| Stack | Function call to function return | Compiler-generated code (automatic, cannot outlive the function) |
| Heap | `malloc` to matching `free` | You, entirely by hand |

The stack is fast (allocating a local variable is just moving a pointer)
but strictly scoped — a pointer to a local variable is garbage the instant
the function returns. The heap is slower (`malloc` has real bookkeeping
work to do) but lives until you explicitly `free` it, which is what makes
it the only option for data that needs to outlive the function that created
it, or whose size isn't known until runtime.

## `realloc`: grow (or shrink) an existing block safely

`realloc` resizes a previously `malloc`'d block, copying the existing
contents if it has to move the block to find enough contiguous space:

```c
// realloc_demo.c -- the correct pattern for growing a heap buffer with realloc
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    size_t count = 4;
    int *arr = malloc(count * sizeof(int));
    for (size_t i = 0; i < count; i++) arr[i] = (int)(i + 1);

    printf("before grow: ");
    for (size_t i = 0; i < count; i++) printf("%d ", arr[i]);
    printf("\n");

    size_t new_count = 8;
    int *tmp = realloc(arr, new_count * sizeof(int));
    if (tmp == NULL) {
        fprintf(stderr, "realloc failed, original block still valid\n");
        free(arr);
        return 1;
    }
    arr = tmp;                       // only overwrite arr once realloc succeeded
    for (size_t i = count; i < new_count; i++) arr[i] = (int)(i + 1);
    count = new_count;

    printf("after grow:  ");
    for (size_t i = 0; i < count; i++) printf("%d ", arr[i]);
    printf("\n");

    free(arr);
    return 0;
}
```

```bash
gcc -Wall -Wextra -g -O0 -fsanitize=address,undefined -o realloc_demo realloc_demo.c
./realloc_demo
```

```text
before grow: 1 2 3 4 
after grow:  1 2 3 4 5 6 7 8
```

The critical detail is the temporary variable: `int *tmp = realloc(arr,
...)` instead of `arr = realloc(arr, ...)`. If `realloc` fails, it returns
`NULL` **and leaves the original block untouched** — overwriting `arr`
directly with that `NULL` would both lose the only pointer to the still-valid
original allocation (an instant leak) and discard your data. Assigning to a
temporary first, checking it, and only then updating `arr` is the only
version of this pattern that handles allocation failure correctly.

## Use-after-free: the bug that only sometimes crashes

```c
// uaf.c -- use-after-free, caught by AddressSanitizer
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int *p = malloc(sizeof(int));
    *p = 42;
    printf("before free: %d\n", *p);

    free(p);
    printf("after free:  %d\n", *p);   // BUG: reads freed memory

    return 0;
}
```

Compiled and run normally, this often prints `42` twice and looks
completely fine — the memory hasn't been reused by anything else yet, so
reading it "works," even though it's undefined behavior. Compiled with
AddressSanitizer, the same bug is caught immediately:

```bash
gcc -Wall -Wextra -g -O0 -fsanitize=address -o uaf uaf.c
./uaf
```

```text
before free: 42
==22402==ERROR: AddressSanitizer: heap-use-after-free on address 0x6020000000d0 at pc 0x000102860928 bp 0x00016d59ea00 sp 0x00016d59e9f8
READ of size 4 at 0x6020000000d0 thread T0
    #0 0x000102860924 in main uaf.c:11
    #1 0x000181d37dfc in start+0x1b4c (dyld:arm64e+0x1fdfc)

0x6020000000d0 is located 0 bytes inside of 4-byte region [0x6020000000d0,0x6020000000d4)
freed by thread T0 here:
    #0 0x0001030cd258 in free+0x7c
    #1 0x0001028608d8 in main uaf.c:10

previously allocated by thread T0 here:
    #0 0x0001030cd164 in malloc+0x78
    #1 0x000102860804 in main uaf.c:6

SUMMARY: AddressSanitizer: heap-use-after-free uaf.c:11 in main
```

ASan poisons freed memory so any later access is caught the instant it
happens, and it prints the allocation site, the free site, *and* the
offending access site — three stack traces that pin down the bug exactly,
instead of a crash somewhere unrelated minutes later once the freed memory
happens to get reused for something else. That "crashes somewhere
unrelated, much later" behavior is precisely why use-after-free bugs are
miserable to find without a tool like this: the symptom and the cause can
be arbitrarily far apart in the program's execution.

**Habit that prevents most of these**: set a pointer to `NULL` immediately
after `free`ing it (`free(p); p = NULL;`). Dereferencing a `NULL` pointer
still crashes, but it crashes *immediately and obviously* at the exact
location of the bug, instead of silently reading whatever now occupies that
freed memory.

## Leaks: memory that's never freed

A leak is simpler than a use-after-free — no crash at all, just memory
that's allocated and never released, so the process's memory footprint
grows for as long as it keeps running. On Linux, `valgrind --leak-check=full
./program` is the standard tool (it runs your program in an instrumented
virtual machine and reports every block still allocated at exit, with the
call stack that allocated it). On macOS, the `leaks` command-line tool
(built into Xcode's command line tools) and Instruments' Leaks template do
the equivalent job. AddressSanitizer's leak detector (`ASAN_OPTIONS=detect_leaks=1`)
covers the same ground on Linux, but is not available on macOS —
know which tool your platform actually supports before you rely on it in
CI.

The fix is always the same regardless of which tool finds it: for every
`malloc`/`calloc`/`realloc` in the program, there must be a reachable path
to exactly one matching `free`. A long-running server (like the one in
Module 10) that leaks even a handful of bytes per request will eventually
exhaust its memory — which is why memory-checking tools are run against
test suites in CI for exactly this kind of code, not just when something
visibly breaks.

## Double-free: freeing the same block twice

Calling `free` twice on the same pointer (without an intervening
`malloc` reusing that address) corrupts the allocator's internal
bookkeeping — the fix is the same discipline as use-after-free: set the
pointer to `NULL` right after freeing it, since `free(NULL)` is explicitly
defined by the C standard to do nothing, making accidental double-frees
harmless as long as you followed that habit:

```c
free(p);
p = NULL;
// ... much later, in a code path that doesn't know p was already freed ...
free(p);   // free(NULL) -- safe no-op, not a crash
```

## Cheat sheet

| Bug | Symptom | Tool that catches it |
|---|---|---|
| Use-after-free | Sometimes silent, sometimes a crash later | ASan (`-fsanitize=address`), valgrind |
| Double-free | Allocator corruption, may crash immediately or later | ASan, valgrind |
| Buffer overflow (heap) | Silent corruption of adjacent data | ASan, valgrind |
| Buffer overflow (stack) | Crash, or a classic security vulnerability | ASan, `-fstack-protector` |
| Memory leak | No crash, just growing memory use | valgrind (Linux), `leaks`/Instruments (macOS) |
| Uninitialized read | Nondeterministic behavior | `-fsanitize=undefined`, valgrind's memcheck |

| Function | What it does | Failure return |
|---|---|---|
| `malloc(n)` | Allocates `n` uninitialized bytes | `NULL` |
| `calloc(n, size)` | Allocates `n * size` bytes, zeroed | `NULL` |
| `realloc(p, n)` | Resizes block `p` to `n` bytes, may move it | `NULL` (original block untouched) |
| `free(p)` | Releases the block; `free(NULL)` is a defined no-op | — |

## Exercise

Take `uaf.c` above and fix it two ways, confirming each with
`-fsanitize=address,undefined`: first, simply move the `free(p)` call to
after the last use of `p`. Second — as a separate version — keep the
original order but add `p = NULL;` immediately after `free(p)`, and add an
`if (p != NULL)` guard around the later access, so the bug becomes
structurally impossible instead of just reordered. Then write a small
program that deliberately leaks a `malloc`'d block inside a loop that runs
1000 times, and run it under `leaks` (macOS) or `valgrind --leak-check=full`
(Linux) to see the tool report exactly how many bytes leaked and where the
allocation happened.
