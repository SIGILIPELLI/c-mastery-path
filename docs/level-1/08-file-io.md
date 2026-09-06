# 08 · File I/O

Programs that forget everything the moment they exit aren't very useful.
C's standard library gives you a small, consistent set of functions —
`fopen`, `fprintf`, `fscanf`, `fgets`, `fclose` — for reading and writing
files, all built around a single type: `FILE *`.

## `FILE *` and `fopen`

`FILE *` is a pointer to a structure the standard library uses internally to
track an open file (its position, buffering, etc.). You never look inside it
yourself — you just pass it to the I/O functions. `fopen` opens a file and
gives you back that pointer, or `NULL` if it failed:

```c
#include <stdio.h>

int main(void) {
    FILE *file = fopen("greeting.txt", "w");   // "w" = write mode

    if (file == NULL) {
        printf("Could not open file for writing.\n");
        return 1;
    }

    fprintf(file, "Hello, file!\n");
    fclose(file);

    printf("Wrote greeting.txt\n");
    return 0;
}
// Output:
// Wrote greeting.txt
```

**Always check the return value of `fopen` before using it.** A missing file,
a bad path, or a permissions problem all make `fopen` return `NULL` — and
dereferencing that `NULL` (by trying to read or write through it) crashes the
program, exactly like the `NULL` pointer risk from
[Module 6](06-pointers-basics.md).

## Modes

The second argument to `fopen` says how you intend to use the file:

| Mode | Meaning | If file doesn't exist | If file exists |
|------|---------|------------------------|-----------------|
| `"r"` | Read (text) | `fopen` returns `NULL` | Read from the start |
| `"w"` | Write (text) | Created | **Overwritten / truncated** |
| `"a"` | Append (text) | Created | Writes are added at the end |
| `"r+"` | Read and write | `fopen` returns `NULL` | Read/write, starts at beginning |
| `"w+"` | Read and write | Created | **Overwritten / truncated** |
| `"a+"` | Read and append | Created | Reads from start, writes append at end |

`"w"` and `"w+"` silently erase any existing content the moment `fopen`
succeeds — reach for `"a"` instead if you want to keep what's already there.

## Writing with `fprintf`

`fprintf` works exactly like `printf`, except the first argument says *where*
to send the output — a `FILE *` instead of the screen:

```c
#include <stdio.h>

int main(void) {
    FILE *file = fopen("numbers.txt", "w");
    if (file == NULL) {
        printf("Error opening file.\n");
        return 1;
    }

    for (int i = 1; i <= 3; i++) {
        fprintf(file, "Line %d: value = %d\n", i, i * i);
    }

    fclose(file);
    printf("Wrote 3 lines to numbers.txt\n");
    return 0;
}
// Output:
// Wrote 3 lines to numbers.txt
```

## Reading lines with `fgets`

For reading text a line at a time, `fgets` is safer than `fscanf` because you
give it a buffer size, so it can't overflow your array:

```c
#include <stdio.h>

int main(void) {
    FILE *file = fopen("numbers.txt", "r");
    if (file == NULL) {
        printf("Error opening file.\n");
        return 1;
    }

    char line[100];
    while (fgets(line, sizeof(line), file) != NULL) {
        printf("Read: %s", line);   // line already includes its own '\n'
    }

    fclose(file);
    return 0;
}
// Output:
// Read: Line 1: value = 1
// Read: Line 2: value = 4
// Read: Line 3: value = 9
```

`fgets` returns `NULL` once it reaches the end of the file, which is exactly
what makes it a natural loop condition — no extra "did I hit the end?" check
needed.

## `fscanf` for structured text

When a file's format is predictable (numbers separated by whitespace, for
example), `fscanf` parses values directly into variables, the same way
`scanf` reads from the keyboard:

```c
#include <stdio.h>

int main(void) {
    FILE *file = fopen("scores.txt", "w");
    fprintf(file, "85 92 78\n");
    fclose(file);

    file = fopen("scores.txt", "r");
    if (file == NULL) {
        printf("Error opening file.\n");
        return 1;
    }

    int a, b, c;
    fscanf(file, "%d %d %d", &a, &b, &c);
    printf("Scores: %d, %d, %d\n", a, b, c);

    fclose(file);
    return 0;
}
// Output:
// Scores: 85, 92, 78
```

## Why `fclose` matters

Closing a file with `fclose` flushes any buffered output to disk and releases
the operating system's file handle. Forgetting to close a file you wrote to
can mean the data never actually makes it to disk (it's stuck in a buffer),
and forgetting to close many files can exhaust the limited number of file
handles a program is allowed to have open at once. Every `fopen` should be
matched with an `fclose`.

## Putting it together: write then read back

```c
#include <stdio.h>

int main(void) {
    // 1. Write a few lines
    FILE *out = fopen("log.txt", "w");
    if (out == NULL) {
        printf("Could not open log.txt for writing.\n");
        return 1;
    }
    fprintf(out, "Startup complete\n");
    fprintf(out, "Loaded 3 modules\n");
    fprintf(out, "Ready\n");
    fclose(out);

    // 2. Read it back and print each line
    FILE *in = fopen("log.txt", "r");
    if (in == NULL) {
        printf("Could not open log.txt for reading.\n");
        return 1;
    }

    char line[100];
    printf("--- log.txt contents ---\n");
    while (fgets(line, sizeof(line), in) != NULL) {
        printf("%s", line);
    }
    fclose(in);

    return 0;
}
// Output:
// --- log.txt contents ---
// Startup complete
// Loaded 3 modules
// Ready
```

## A preview of binary I/O

Everything above is **text** I/O — human-readable, but not the most compact
or efficient way to store structured data like a whole `struct` at once. C
also has `fread` and `fwrite` for **binary** I/O, which copy raw bytes
directly (an entire `struct Point`, for example, in one call) instead of
formatting them as text. Binary I/O is covered in depth in
[Level 2, Module 5](../level-2/05-binary-file-io.md), once you've seen more of
what structs (from [Module 7](07-structs.md)) can hold.

## How It Actually Works

`fopen` is a thin wrapper around your operating system's own file-opening
mechanism — on Linux/macOS, ultimately the `open()` system call, which asks
the kernel to locate the file on disk, check permissions, and hand back a
small integer called a **file descriptor** that the kernel uses internally
to track the open file (its current read/write position, buffering state,
and so on). The `FILE *` you get back from `fopen` is a heap-allocated
structure — defined by the C standard library, not the kernel — that wraps
that file descriptor together with a user-space buffer, which is why
`FILE *` is opaque: you're not meant to know or rely on its internal
layout, only pass it to `stdio.h` functions.

That user-space buffer is the whole reason `fclose` matters. `fprintf`
doesn't write to disk immediately — for efficiency, it first appends bytes
into an in-memory buffer inside the `FILE` structure (typically several
kilobytes), and only flushes that buffer to the kernel (via a `write()`
system call) when it fills up, when you call `fflush`, or when you call
`fclose`. If the program crashes or exits abnormally before the buffer is
flushed, whatever was sitting in that buffer never reaches the disk — the
data existed only in the process's own memory, not yet handed off to the
kernel's page cache. `fclose` both flushes this buffer and releases the
file descriptor back to the OS, which matters because a process is only
allowed a limited number of open file descriptors at once (commonly 1024)
— leaking them by skipping `fclose` in a long-running program eventually
makes every further `fopen` fail.

The different **modes** (`"r"`, `"w"`, `"a"`) map to flags passed straight
through to the kernel's `open()` call — `"w"` corresponds to
`O_WRONLY | O_CREAT | O_TRUNC`, where `O_TRUNC` is what causes the kernel
to immediately discard the file's existing contents and reset its size to
zero the moment the file is opened, before you've written a single byte —
which is why the truncation happens even if your program crashes right
after `fopen` and never calls `fprintf` at all.

`fgets` reading in a loop until `NULL` works because internally each call
asks the buffered layer for the next chunk up to (or including) a newline;
when the underlying `read()` system call returns zero bytes (end of file),
`fgets` has nothing left to return and reports that with `NULL` — there's
no separate "are we at EOF" flag you have to poll, the return value itself
carries that information because the kernel told the library the file is
exhausted.

## Exercise

Write a program that asks the user (with `scanf`) for 5 integers, one at a
time, and writes each one to a file called `input.txt`, one per line. Then,
in the same program, reopen `input.txt` for reading, read all 5 numbers back
with `fscanf` (or `fgets` + parsing), compute their sum and average, and
print both. Make sure to check every `fopen` call for `NULL` before using it.
