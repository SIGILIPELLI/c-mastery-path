# 09 · Debugging Tools

C gives you no safety net. A one-character typo — `<=` instead of `<`, `=`
instead of `==`, a forgotten `free` — can compile cleanly and still crash, or
worse, silently corrupt memory and keep running. `printf` debugging gets you
only so far. This module covers the two tools that actually earn their keep:
**gdb**, which lets you pause a running program and inspect it, and
**valgrind**, which catches memory bugs that don't crash *today* but will in
production.

## Building for debugging

Nothing useful shows up in gdb without debug symbols:

```bash
gcc -Wall -Wextra -g -O0 -o app main.c
```

- `-g` embeds source-line and variable-name information in the binary.
- `-O0` disables optimization — optimized code reorders and eliminates
  variables, which makes stepping through it in a debugger confusing or
  outright misleading. Debug locally at `-O0`; optimize for release.

## gdb: stepping through a running program

Consider a deliberately broken program:

```c
// buggy.c
#include <stdio.h>

int sum_range(int low, int high) {
    int total = 0;
    for (int i = low; i <= high; i++) {
        total += i;
    }
    return total;
}

int main(void) {
    int values[5] = {10, 20, 30, 40, 50};
    int result = sum_range(0, 5);   // BUG: should be 0, 4 -- reads values[5], out of bounds
    printf("Sum of first 5: %d\n", result);
    printf("Values: %d %d %d %d %d\n",
           values[0], values[1], values[2], values[3], values[4]);
    return 0;
}
```

This compiles and often *appears* to run fine — the out-of-bounds read is
undefined behaviour, not a guaranteed crash. That's exactly the kind of bug
that needs a debugger rather than guesswork.

```bash
gcc -Wall -Wextra -g -O0 -o buggy buggy.c
gdb ./buggy
```

### The core gdb workflow

```text
(gdb) break main
Breakpoint 1 at 0x1169: file buggy.c, line 13.
(gdb) run
Starting program: ./buggy

Breakpoint 1, main () at buggy.c:13
13          int values[5] = {10, 20, 30, 40, 50};
(gdb) next
14          int result = sum_range(0, 5);
(gdb) step
sum_range (low=0, high=5) at buggy.c:5
5           int total = 0;
(gdb) print low
$1 = 0
(gdb) print high
$2 = 5
```

| Command | What it does |
|---------|---------------|
| `break <function>` / `break <file>:<line>` | Set a breakpoint |
| `run` | Start the program (stops at breakpoints) |
| `next` | Execute the current line; **step over** function calls |
| `step` | Execute the current line; **step into** function calls |
| `continue` | Resume until the next breakpoint or exit |
| `print <expr>` | Evaluate and print a variable or expression |
| `backtrace` (or `bt`) | Show the call stack |
| `list` | Show source around the current line |
| `watch <var>` | Break whenever `<var>`'s value changes |
| `quit` | Exit gdb |

Spotting the bug: set a breakpoint inside the loop and watch `i` and `total`
evolve.

```text
(gdb) break buggy.c:7
Breakpoint 2 at 0x1145: file buggy.c, line 7.
(gdb) continue
Breakpoint 2, sum_range (low=0, high=5) at buggy.c:7
7               total += i;
(gdb) print i
$3 = 5
```

`i` reaches `5`, but `values` only has indices `0`-`4`. The call site passed
the wrong `high`. `next` through a few more iterations and `backtrace` to see
which frame called `sum_range` with the bad argument — that's the actual fix
location, not the loop itself.

### When the program has already crashed: core dumps

For a real segfault, you don't need to reproduce it interactively — gdb can
load the crash state directly.

```bash
ulimit -c unlimited          # allow core dumps in this shell
./crashy                     # Segmentation fault (core dumped)
gdb ./crashy core
```

```text
(gdb) backtrace
#0  0x0000555555555149 in append_char (s=0x0, c=65 'A') at crashy.c:4
#1  0x0000555555555171 in main () at crashy.c:12
```

Frame `#0` is exactly where it died, and `s=0x0` tells you the whole story: a
null pointer was passed in. `backtrace` is often the single most useful gdb
command — it answers "where did this actually happen" immediately, without
any manual stepping.

## valgrind: catching memory bugs that don't crash

Some bugs never segfault — they read one byte past an array, or leak memory a
few bytes at a time, and the program runs "fine" for years until it doesn't.
`valgrind`'s `memcheck` tool (the default) runs your program in an instrumented
virtual machine that tracks every allocation, free, and memory access.

```c
// leaky.c
#include <stdlib.h>
#include <string.h>

char *make_greeting(const char *name) {
    char *greeting = malloc(20);           // BUG: too small for long names
    strcpy(greeting, "Hello, ");
    strcat(greeting, name);                // may overflow the 20 bytes
    return greeting;                       // BUG: caller never frees this
}

int main(void) {
    char *msg = make_greeting("Alexandria Constantinople");   // long enough to overflow
    (void)msg;                             // pretend we used it
    return 0;
}
```

```bash
gcc -Wall -Wextra -g -O0 -o leaky leaky.c
valgrind --leak-check=full ./leaky
```

```text
==12345== Invalid write of size 1
==12345==    at 0x4849C29: strcat (vg_replace_strmem.c:...)
==12345==    by 0x1091A5: make_greeting (leaky.c:8)
==12345==    by 0x1091E2: main (leaky.c:13)
==12345==  Address 0x4a4d054 is 0 bytes after a block of size 20 alloc'd
==12345==    at 0x484D9C4: malloc (vg_replace_malloc.c:...)
==12345==    by 0x109188: make_greeting (leaky.c:6)
...
==12345== HEAP SUMMARY:
==12345==     in use at exit: 21 bytes in 1 blocks
==12345==   total heap usage: 1 allocs, 0 frees, 21 bytes allocated
==12345==
==12345== 21 bytes in 1 blocks are definitely lost in loss record 1 of 1
==12345==    at 0x484D9C4: malloc (vg_replace_malloc.c:...)
==12345==    by 0x109188: make_greeting (leaky.c:6)
==12345==    by 0x1091E2: main (leaky.c:13)
```

Two separate defects, both caught precisely: the `Invalid write` pins the
overflow to `leaky.c:8`, and `definitely lost` pins the missing `free` to
`leaky.c:6` — the exact `malloc` call whose return value was never released.

| Valgrind verdict | Meaning |
|-------------------|---------|
| `Invalid read/write of size N` | Accessed memory outside an allocated block |
| `Use of uninitialised value` | Read a variable before writing to it |
| `Invalid free() / delete` | Freed a pointer twice, or one never returned by `malloc` |
| `definitely lost` | Leaked memory with no remaining pointer to it — a real leak |
| `still reachable` | Never freed, but a pointer to it still exists at exit — often OS-cleaned buffers, lower priority |

### Reading `--leak-check=full` output

Run it with `-s` for a summary even on success:

```bash
valgrind --leak-check=full --show-leak-kinds=all -s ./app
```

A clean run ends with:

```text
==12345== HEAP SUMMARY:
==12345==     in use at exit: 0 bytes in 0 blocks
==12345==   total heap usage: 4 allocs, 4 frees, 1,024 bytes allocated
==12345==
==12345== All heap blocks were freed -- no leaks are possible
==12345==
==12345== ERROR SUMMARY: 0 errors from 0 contexts
```

That `0 errors from 0 contexts` line is what you want to see before shipping
anything that touches `malloc`.

## Sanitizers: catching the same bugs faster

Valgrind is thorough but slow (10-50× slowdown). For day-to-day development,
compiling with sanitizers built into gcc/clang catches most of the same bugs
at nearly full speed:

```bash
gcc -Wall -Wextra -g -O0 -fsanitize=address,undefined -o app main.c
./app
```

```text
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x...
WRITE of size 1 at 0x... thread T0
    #0 0x... in strcat
    #1 0x... in make_greeting leaky.c:8
    #2 0x... in main leaky.c:13
```

Same bug, same line number, and it fires the moment the overflow happens —
no separate tool invocation needed. A reasonable workflow: build with
sanitizers by default during development (they're what
[Module 8](08-makefiles-build-systems.md)'s `BUILD=debug` branch already
enables), and run the full `valgrind` sweep before a release or when a
sanitizer report is confusing.

## Which tool for which bug

| Symptom | Reach for |
|---------|-----------|
| "It crashes, where?" | `gdb` + `backtrace` (with a core dump if it already crashed) |
| "What value does this variable have right here?" | `gdb` with a breakpoint and `print` |
| "Is this leaking memory?" | `valgrind --leak-check=full` |
| "Am I reading/writing out of bounds?" | `-fsanitize=address` (fast) or `valgrind` (thorough) |
| "Did I read a variable before initializing it?" | `valgrind` (`Use of uninitialised value`) or `-fsanitize=undefined` |

## Exercise

Take the `leaky.c` example above.

1. Compile it with `-g -O0` and run it under `valgrind --leak-check=full`.
   Confirm you see both the invalid write and the "definitely lost" block, and
   note the exact line numbers valgrind reports.
2. Fix the size bug (allocate enough for `"Hello, "` + the name + the null
   terminator — `strlen(name) + 8` is the right shape) and the leak (free the
   result in `main`, or better, decide who owns the returned pointer and
   document it in a comment).
3. Re-run valgrind and confirm `0 errors from 0 contexts` and
   `All heap blocks were freed`.
4. Recompile with `-fsanitize=address,undefined` instead and confirm it
   reports the same overflow (before your fix) and is silent (after).
5. Deliberately introduce a **double free** (`free(msg); free(msg);`) and run
   both valgrind and the sanitizer build against it. Compare how clearly each
   one identifies the problem.
