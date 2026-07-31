# 06 · Error Handling Conventions

C has no exceptions. There is no `try`, no `catch`, no stack unwinding — when
something fails, the function must *tell you*, and you must *ask*. That sounds
primitive, and it is, but it also means error handling in C is completely
explicit: you can see every failure path in the source.

The catch is that C's conventions are inconsistent by historical accident.
`malloc` returns `NULL` on failure. `fopen` returns `NULL` too, but also sets
`errno`. `printf` returns a negative number. `main` returns `0` for success
while most predicate functions return non-zero for true. This module maps the
conventions and shows how to write code that handles failure cleanly.

## The dominant convention: return a status, use out-parameters for data

```c
#include <stdio.h>
#include <stdlib.h>

// Returns 1 on success, 0 on failure. The VALUE comes back through *out.
int parse_positive_int(const char *text, int *out) {
    if (text == NULL || out == NULL) return 0;

    char *end;
    long value = strtol(text, &end, 10);

    if (end == text)     return 0;   // no digits at all
    if (*end != '\0')    return 0;   // trailing junk: "12abc"
    if (value <= 0)      return 0;   // not positive
    if (value > 100000)  return 0;   // out of our accepted range

    *out = (int)value;
    return 1;
}

int main(void) {
    const char *inputs[] = {"42", "abc", "12x", "-5", "999999"};

    for (int i = 0; i < 5; i++) {
        int value;
        if (parse_positive_int(inputs[i], &value)) {
            printf("%-8s -> %d\n", inputs[i], value);
        } else {
            printf("%-8s -> rejected\n", inputs[i]);
        }
    }
    return 0;
}
// Output:
// 42       -> 42
// abc      -> rejected
// 12x      -> rejected
// -5       -> rejected
// 999999   -> rejected
```

Why not just return the parsed number and use `-1` for failure? Because `-1` is
a *valid* integer. Reserving a sentinel value only works when your result range
genuinely excludes it. The status-plus-out-parameter shape always works, which
is why it's the most common design in real C APIs.

This is also why `strtol` exists at all. `atoi("abc")` returns `0` — completely
indistinguishable from `atoi("0")`. **Never use `atoi` for input you don't
control.**

| Return style | Example | Good when |
|--------------|---------|-----------|
| `NULL` on failure | `malloc`, `fopen`, `strchr` | the result is a pointer |
| `0` success / `-1` failure | `open`, `close`, `stat` (POSIX) | no value to return |
| `1` success / `0` failure | your own predicates | reads naturally in `if` |
| Negative on failure | `printf`, `snprintf` | a non-negative count is the result |
| Enum error code | `enum { OK, ERR_IO, ERR_RANGE }` | callers need to distinguish causes |

## `errno` — the standard library's error detail

Many standard and POSIX functions set the global `errno` when they fail. It
answers *why*, once you already know *that*:

```c
#include <stdio.h>
#include <errno.h>
#include <string.h>

int main(void) {
    errno = 0;                       // clear it before the call

    FILE *f = fopen("/definitely/not/here.txt", "r");
    if (f == NULL) {
        // Two ways to report the same thing:
        printf("errno = %d: %s\n", errno, strerror(errno));
        perror("fopen");             // prints "fopen: <message>" to stderr
        return 1;
    }

    fclose(f);
    return 0;
}
// Output:
// errno = 2: No such file or directory
// fopen: No such file or directory
```

Three rules that prevent most `errno` bugs:

1. **Only read `errno` after a call has already indicated failure.** A
   successful call may set `errno` to anything; the standard does not require it
   to be left alone.
2. **Read it immediately.** Any intervening library call — including the
   `printf` you use to report the error — can overwrite it. Save it to a local
   if you need it later.
3. **Set `errno = 0` first** when a function signals failure ambiguously
   (`strtol` returning `LONG_MAX` could be a real value or an overflow).

```c
#include <errno.h>
#include <limits.h>
#include <stdlib.h>
#include <stdio.h>

int main(void) {
    errno = 0;
    long v = strtol("99999999999999999999", NULL, 10);

    if (errno == ERANGE) {
        printf("overflow -- strtol clamped to %ld\n", v);
    }
    return 0;
}
// Output:
// overflow -- strtol clamped to 9223372036854775807
```

| Common `errno` value | Meaning |
|----------------------|---------|
| `ENOENT` | No such file or directory |
| `EACCES` | Permission denied |
| `ENOMEM` | Out of memory |
| `EINVAL` | Invalid argument |
| `ERANGE` | Result out of range (numeric overflow) |
| `EEXIST` | File already exists |

`strerror(errno)` turns any of these into a human-readable string;
`perror("context")` prints `context: message` to `stderr` in one call.

## Your own error codes

When callers need to react differently to different failures, an enum beats a
bare `0`/`1`:

```c
#include <stdio.h>

typedef enum {
    CFG_OK = 0,
    CFG_ERR_NOT_FOUND,
    CFG_ERR_BAD_SYNTAX,
    CFG_ERR_OUT_OF_MEMORY
} ConfigError;

const char *config_error_string(ConfigError e) {
    switch (e) {
        case CFG_OK:                 return "success";
        case CFG_ERR_NOT_FOUND:      return "config file not found";
        case CFG_ERR_BAD_SYNTAX:     return "malformed line in config";
        case CFG_ERR_OUT_OF_MEMORY:  return "out of memory";
    }
    return "unknown error";
}

int main(void) {
    ConfigError err = CFG_ERR_BAD_SYNTAX;

    if (err != CFG_OK) {
        fprintf(stderr, "config: %s (code %d)\n",
                config_error_string(err), err);
        return 1;
    }
    return 0;
}
// Output (on stderr):
// config: malformed line in config (code 2)
```

Making `CFG_OK` equal `0` lets callers write `if (err)` naturally, and the
`switch` without a `default` means the compiler warns you if you add a new enum
value and forget to describe it.

## Cleaning up on failure: the `goto` ladder

This is the one place idiomatic C uses `goto`, and it is genuinely the clearest
option. When a function acquires several resources, every failure path has to
release exactly the ones acquired so far:

```c
#include <stdio.h>
#include <stdlib.h>

int copy_file(const char *src, const char *dst) {
    FILE *in = NULL, *out = NULL;
    char *buf = NULL;
    int result = 0;                       // 0 = failure, 1 = success

    in = fopen(src, "rb");
    if (in == NULL) { perror(src); goto cleanup; }

    out = fopen(dst, "wb");
    if (out == NULL) { perror(dst); goto cleanup; }

    buf = malloc(4096);
    if (buf == NULL) { fprintf(stderr, "out of memory\n"); goto cleanup; }

    size_t n;
    while ((n = fread(buf, 1, 4096, in)) > 0) {
        if (fwrite(buf, 1, n, out) != n) {
            perror("write");
            goto cleanup;
        }
    }
    if (ferror(in)) { perror("read"); goto cleanup; }

    result = 1;                           // made it all the way through

cleanup:
    free(buf);                            // free(NULL) is safe
    if (out) fclose(out);
    if (in)  fclose(in);
    return result;
}

int main(void) {
    if (!copy_file("source.txt", "dest.txt")) {
        fprintf(stderr, "copy failed\n");
        return 1;
    }
    printf("copied\n");
    return 0;
}
```

Every resource is initialized to `NULL`, there is exactly **one** exit point,
and the cleanup block is safe to run no matter how far the function got. The
alternative — freeing the right subset at each of five `return` statements — is
where leaks come from.

Note `if (ferror(in))` after the loop. `fread` returning 0 could mean EOF *or* a
read error; `feof()` and `ferror()` tell you which. Assuming success is how
silent data corruption happens.

## Reporting: `stderr`, exit codes, and `assert`

```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

double average(const int *values, int n) {
    // assert documents an assumption the CALLER must satisfy.
    // Compiled out entirely when NDEBUG is defined (gcc -DNDEBUG).
    assert(values != NULL);
    assert(n > 0);

    long sum = 0;
    for (int i = 0; i < n; i++) sum += values[i];
    return (double)sum / n;
}

int main(int argc, char **argv) {
    if (argc < 2) {
        // Usage errors go to stderr, not stdout, so pipes stay clean.
        fprintf(stderr, "usage: %s <file>\n", argv[0]);
        return EXIT_FAILURE;      // conventionally 1
    }

    int data[] = {4, 8, 15, 16, 23, 42};
    printf("average = %.2f\n", average(data, 6));

    return EXIT_SUCCESS;          // 0
}
// Output:
// average = 18.00
```

The division of labour matters:

- **`assert`** is for *programmer* errors — conditions that should be impossible
  if your code is correct. It aborts the process and is removed in release
  builds, so never put anything with side effects inside it, and never use it to
  validate user input.
- **Return codes / `errno`** are for *runtime* errors — a missing file, a full
  disk, bad user input. These are expected and must be handled.
- **`stderr` vs `stdout`**: diagnostics go to `stderr` so that
  `./prog > results.txt` keeps error messages visible on the terminal.
- **`exit(EXIT_FAILURE)`** or `return EXIT_FAILURE` from `main` tells the shell
  something went wrong, which is what `&&` and CI pipelines check.

## Exercise

Write a small line-counting utility with proper error handling. Define
`typedef enum { FC_OK, FC_ERR_OPEN, FC_ERR_READ } FileCountError;` and a
function `FileCountError count_lines(const char *path, long *out_lines)` that
opens the file, counts `'\n'` characters with `fgetc`, checks `ferror` before
returning, and always closes the file — using the `goto cleanup` pattern. In
`main`, take the filename from `argv[1]`, print a usage message to `stderr` and
return `EXIT_FAILURE` if it's missing, and on failure print both your enum's
message and `strerror(errno)`. Test it against a real file, a nonexistent file,
and a directory (which opens differently than you might expect).
