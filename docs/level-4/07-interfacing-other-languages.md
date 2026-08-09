# 07 · Interfacing with Other Languages

C is the lingua franca of the software world. Python, Ruby, Rust, Go, Java,
C#, Lua and JavaScript can all call C, and almost none of them can call each
other directly. The reason is the **C ABI** — a stable, documented agreement
about how arguments go into registers, how the stack is laid out, and what a
symbol name looks like. C has no name mangling, no runtime, no garbage
collector, and no exceptions, so there is very little to disagree about.

That is why "write the fast part in C" works. This module builds a C library
and calls it from Python with `ctypes`, which needs no compilation on the
Python side and shows the boundary issues plainly.

## A library designed to be called

The API shape matters more than the algorithms. Every function here is
chosen to demonstrate one boundary problem:

```c
// stats.c -- a C library designed to be called from another language
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>

typedef struct { double mean, stddev; int count; } Summary;

/* Plain scalars: the easy case. */
int add(int a, int b) { return a + b; }

/* Array in, struct out by pointer -- caller owns the struct. */
void summarize(const double *data, int n, Summary *out) {
    double sum = 0.0;
    for (int i = 0; i < n; i++) sum += data[i];
    out->count = n;
    out->mean  = n ? sum / n : 0.0;

    double sq = 0.0;
    for (int i = 0; i < n; i++) { double d = data[i] - out->mean; sq += d * d; }
    out->stddev = n ? sqrt(sq / n) : 0.0;
}

/* Array in, array out IN PLACE -- no ownership question at all. */
void scale(double *data, int n, double factor) {
    for (int i = 0; i < n; i++) data[i] *= factor;
}

/* C allocates a string. The caller MUST call free_string. */
char *describe(const Summary *s) {
    char *buf = malloc(128);
    if (!buf) return NULL;
    snprintf(buf, 128, "n=%d mean=%.3f sd=%.3f", s->count, s->mean, s->stddev);
    return buf;
}

void free_string(char *p) { free(p); }

/* A callback: C calls back into the host language. */
typedef int (*Filter)(double);

int count_matching(const double *data, int n, Filter pred) {
    int c = 0;
    for (int i = 0; i < n; i++) if (pred(data[i])) c++;
    return c;
}
```

Build it as a shared library — the same `-fPIC -shared` from
[Level 3 module 06](../level-3/06-static-dynamic-libraries.md):

```bash
clang -Wall -Wextra -std=c11 -fPIC -shared -o libstats.dylib stats.c
nm -gU libstats.dylib
```

```text
0000000000000408 T _add
00000000000006a8 T _count_matching
00000000000005f0 T _describe
0000000000000684 T _free_string
0000000000000590 T _scale
0000000000000428 T _summarize
```

Six plain, unmangled names. That listing *is* the ABI — the other language
looks up exactly these strings. (C++ would show
`_Z3addii`-style mangled names, which is why C++ libraries meant for FFI
wrap their entry points in `extern "C"`.)

## Calling it from Python

```python
# use_stats.py -- calling libstats from Python with ctypes
import ctypes

lib = ctypes.CDLL("./libstats.dylib")   # .so on Linux, .dll on Windows

class Summary(ctypes.Structure):
    _fields_ = [("mean",   ctypes.c_double),
                ("stddev", ctypes.c_double),
                ("count",  ctypes.c_int)]

# Declaring signatures is NOT optional -- see the next section.
lib.add.argtypes = [ctypes.c_int, ctypes.c_int]
lib.add.restype  = ctypes.c_int

lib.summarize.argtypes = [ctypes.POINTER(ctypes.c_double), ctypes.c_int,
                          ctypes.POINTER(Summary)]
lib.summarize.restype  = None

lib.describe.argtypes    = [ctypes.POINTER(Summary)]
lib.describe.restype     = ctypes.POINTER(ctypes.c_char)   # NOT c_char_p
lib.free_string.argtypes = [ctypes.POINTER(ctypes.c_char)]

FILTER = ctypes.CFUNCTYPE(ctypes.c_int, ctypes.c_double)
lib.count_matching.argtypes = [ctypes.POINTER(ctypes.c_double), ctypes.c_int, FILTER]
lib.count_matching.restype  = ctypes.c_int

values = [2.0, 4.0, 4.0, 4.0, 5.0, 5.0, 7.0, 9.0]
buf = (ctypes.c_double * len(values))(*values)      # a real C double[8]

s = Summary()
lib.summarize(buf, len(values), ctypes.byref(s))

lib.scale(buf, len(buf), 10.0)

ptr = lib.describe(ctypes.byref(s))
print("describe ->", ctypes.cast(ptr, ctypes.c_char_p).value.decode())
lib.free_string(ptr)          # C allocated it, C must free it

@FILTER
def is_big(x):
    return 1 if x > 4.5 else 0

print("count_matching ->", lib.count_matching(buf, len(buf), is_big))
```

```bash
python3 use_stats.py
```

```text
add(2, 40)        = 42
summarize         -> count=8 mean=5.0 stddev=2.0
scale(x10)        -> [20.0, 40.0, 40.0, 40.0] ...
describe          -> n=8 mean=5.000 sd=2.000
count_matching    -> 8 of 8 (after scaling, all are > 4.5)
```

Everything crossed the boundary: scalars, an array modified in place, a
struct filled by pointer, a heap string, and a Python function called *from*
C once per element.

`(ctypes.c_double * len(values))(*values)` is the important line. It builds
an actual contiguous C `double[8]` — a Python list would not do, because a
list of floats is an array of pointers to boxed objects with no contiguous
doubles anywhere in it.

## Why declaring signatures is mandatory

`ctypes` will happily call a function you have not described. It assumes
every argument is passed as-is and **every return value is a C `int`**.
Here is what that costs:

```python
# trap2.py -- the same function, declared and undeclared
lib.describe.restype = ctypes.POINTER(ctypes.c_char)
good = lib.describe(ctypes.byref(s))
true_addr = ctypes.cast(good, ctypes.c_void_p).value

# Now the default: ctypes assumes every function returns C int.
raw = lib2.describe(ctypes.byref(s))

buf = (ctypes.c_double * 3)(1.0, 2.0, 3.0)
lib2.scale(buf, 3, 10)          # 10 is a Python int, not 10.0
```

```text
true pointer          : 0x0000000151f127e0
default restype (int) : 0x0000000051f127e0  <- truncated to 32 bits
dereferencing that address would read unmapped memory.
scale(buf, 3, 10) with no argtypes -> [0.0, 0.0, 0.0]
same call WITH argtypes            ->  [10.0, 20.0, 30.0]
```

Two catastrophes on one screen. The 64-bit pointer came back truncated to
its low 32 bits — `0x151f127e0` became `0x51f127e0`, an address that is not
mapped and not even the same object. And `scale` with an integer `10` where
a `double` was expected produced **zeros**: on this ABI, floating-point
arguments travel in floating-point registers, so the integer went into a
general-purpose register and `scale` read whatever garbage was in `d0`.

Neither raised an exception. Neither printed a warning. Both are silent
corruption of exactly the kind [module 05](05-security-in-c.md) is about,
except now the bug is in the glue code rather than the C.

**Declare `argtypes` and `restype` for every function you call.** Without
them you are not calling C, you are guessing at an ABI.

## Ownership across the boundary

The hardest part of FFI is not types, it is lifetimes. Neither side's
memory model applies to the other: Python's garbage collector does not know
about `malloc`, and `free` does not know about reference counts.

The rule that makes this tractable: **whoever allocates, frees**, and the
API must expose a way to do it. That is why `describe` has a matching
`free_string`, and why the Python code declares `restype` as
`POINTER(c_char)` rather than the convenient `c_char_p`:

```python
lib.describe.restype = ctypes.c_char_p        # the tempting shortcut
p = lib.describe(ctypes.byref(s))             # p is a Python bytes COPY
```

```text
describe as c_char_p -> b'n=4 mean=2.500 sd=1.118'
...but we now hold a Python bytes copy and have LOST the C pointer,
   so free_string() can never be called on it: a guaranteed leak.
```

`c_char_p` silently copies the string into a Python `bytes` object and
discards the original pointer. The data is correct and the 128-byte
allocation is unreachable forever. Every call leaks.

The four patterns, in order of preference:

| Pattern | Ownership | Example |
|---|---|---|
| Caller provides the buffer | Caller's, always | `summarize(data, n, &out)` |
| Modify in place | Caller's, always | `scale(data, n, f)` |
| C allocates + explicit free | C's, released by the caller | `describe` / `free_string` |
| C returns a pointer to static/internal data | C's, never freed | Must document invalidation rules |

Prefer the first two. A function that fills a caller-supplied buffer has no
ownership question to get wrong, which is why so many C APIs take a
`(buf, size)` pair and return the length they needed.

Callbacks have their own lifetime trap. The `@FILTER`-decorated Python
function must be kept alive by a Python reference for as long as C might
call it — assigning it to a local that goes out of scope while C still holds
the pointer is a use-after-free with a Python object on the other end.
Callbacks must also **never let an exception escape** into C: C has no
unwinding mechanism, so the stack is corrupted rather than unwound. Catch
everything inside the callback and signal failure with a return value.

## Cheat sheet

| Concern | Rule |
|---|---|
| Symbol names | C only. C++ needs `extern "C"` to avoid mangling |
| Build | `-fPIC -shared`; `.so` (Linux), `.dylib` (macOS), `.dll` (Windows) |
| Types at the boundary | Only `<stdint.h>` fixed-width types, `double`, and pointers |
| `int` / `long` | Avoid — width varies by platform ([module 04](04-writing-portable-c.md)) |
| Structs | Layout must match exactly; padding is implementation-defined |
| Strings | C: null-terminated bytes. Encode/decode explicitly, never assume UTF-8 |
| Errors | Return codes or an out-parameter. Never exceptions, never `longjmp` |
| Ownership | Whoever allocates, frees. Export a `free_*` for anything you return |
| Callbacks | Keep the host-side object alive; never let exceptions escape |
| Threads | Document whether your functions are thread-safe; the host may call from many |

| Binding tool | Notes |
|---|---|
| Python `ctypes` | Stdlib, no build step, runtime-only checking — used here |
| Python `cffi` | Parses real C declarations; safer, needs installing |
| Python C API | Fastest, most work, ties you to CPython versions |
| Rust | `extern "C"` + `#[repr(C)]`; `bindgen` generates declarations from headers |
| Go | `cgo`; note the real cost per call and the pointer-passing rules |
| Java | JNI (verbose) or the newer FFM API (`java.lang.foreign`) |
| Node.js | N-API |

## Exercise

Wrap the key-value store from
[Level 3's project](../level-3/10-project-kv-store.md) as a shared library
and drive it from Python. Export `kv_create`, `kv_set`, `kv_get`,
`kv_delete`, `kv_count`, `kv_destroy`, plus a `kv_free_value` so the string
returned by `kv_get` can be released. Write a Python class that wraps the
opaque handle, declares every signature, encodes/decodes strings explicitly,
and frees the C string in a `finally` block.

Then prove the two things that are easy to get wrong. First, run it under
`-fsanitize=address` (via `DYLD_INSERT_LIBRARIES`, or `LD_PRELOAD` on Linux)
and confirm a full create/populate/destroy cycle leaks nothing — then
deliberately drop the `kv_free_value` call and confirm the leak is reported.
Second, call the same store from four Python threads at once and verify the
`pthread_rwlock` still holds up; then explain why Python's GIL does **not**
protect your C code, and what would happen if the store were not internally
locked.
