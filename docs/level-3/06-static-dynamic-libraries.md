# 06 · Static & Dynamic Libraries

[Module 07 of Level 2](../level-2/07-modular-programming.md) split code
across `.c`/`.h` files compiled together into one binary. A **library**
takes that one step further: compiled code that other programs — including
ones you didn't write and won't ever see the source of — can link against
without recompiling it. C has two kinds, built and used differently enough
that picking the right one is a real design decision, not just a build
detail.

## The library under test

Everything below builds the same tiny library — two functions, one header:

```c
// mathutils.h
#ifndef MATHUTILS_H
#define MATHUTILS_H

int add(int a, int b);
int square(int n);

#endif
```

```c
// mathutils.c
#include "mathutils.h"

int add(int a, int b) {
    return a + b;
}

int square(int n) {
    return n * n;
}
```

```c
// main.c
#include <stdio.h>
#include "mathutils.h"

int main(void) {
    printf("add(2, 3) = %d\n", add(2, 3));
    printf("square(6) = %d\n", square(6));
    return 0;
}
```

## Static libraries: copied into the binary at link time

A static library (`.a` on Linux/macOS, `.lib` on Windows) is just an
**archive of `.o` object files**. The linker copies whatever functions the
program actually calls straight into the final executable — once linked,
the library isn't needed again.

```bash
gcc -Wall -Wextra -c mathutils.c -o mathutils.o     # compile to object file
ar rcs libmathutils.a mathutils.o                    # archive it
gcc -Wall -Wextra -o app_static main.c -L. -lmathutils
./app_static
```

```text
add(2, 3) = 5
square(6) = 36
```

`ar rcs` creates (`c`) the archive, inserts with replace (`r`), and builds
an index (`s`) so the linker can look up symbols without scanning every
member. `-L.` tells the linker to also search the current directory for
libraries; `-lmathutils` tells it to look for a file named
`libmathutils.a` (the `lib` prefix and `.a` suffix are added automatically
— you never write them in the flag). You can confirm what actually ended up
inside the archive with `nm`:

```bash
nm libmathutils.a
```

```text
mathutils.o:
0000000000000000 T _add
0000000000000020 T _square
```

`T` marks a symbol defined in the text (code) section — both functions are
present and ready for the linker to pull in.

**Tradeoff**: `app_static` runs standalone with no runtime dependency on
`libmathutils.a` at all — copy the executable anywhere and it works. The
cost is size (every program that links the library gets its own copy of the
code) and staleness (fixing a bug in the library means recompiling and
relinking every program that uses it).

## Dynamic (shared) libraries: loaded at runtime

A dynamic library (`.so` on Linux, `.dylib` on macOS — this walkthrough
uses `.so` since clang accepts it as a name on both, but macOS tooling
conventionally expects `.dylib`) stays a **separate file** that the
operating system loads into memory once and maps into every process that
needs it, at the moment each process starts (or later, with `dlopen`):

```bash
gcc -Wall -Wextra -fPIC -c mathutils.c -o mathutils_pic.o
gcc -shared -o libmathutils.so mathutils_pic.o
gcc -Wall -Wextra -o app_dynamic main.c -L. -lmathutils
./app_dynamic
```

```text
add(2, 3) = 5
square(6) = 36
```

`-fPIC` compiles **position-independent code** — machine code that doesn't
hardcode absolute memory addresses, so the same compiled bytes work no
matter where the OS happens to map the library into a given process's
address space. Static libraries don't need this because their code gets
copied into one fixed binary at link time; shared libraries are mapped into
a different address in every process that loads them, so PIC is mandatory.

Inspect what `app_dynamic` actually depends on at runtime:

```bash
otool -L app_dynamic     # macOS; use `ldd app_dynamic` on Linux
```

```text
app_dynamic:
	libmathutils.so (compatibility version 0.0.0, current version 0.0.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1356.0.0)
```

Unlike the static build, `libmathutils.so` is **not** inside `app_dynamic`
— it's a name the OS's dynamic loader resolves at startup. Move
`app_dynamic` to a directory that doesn't have `libmathutils.so` sitting
next to (or reachable from) it, and loading fails outright, before `main`
even runs:

```bash
cp app_dynamic /tmp/ && cd /tmp && ./app_dynamic
```

```text
dyld[23504]: Library not loaded: libmathutils.so
  Referenced from: <...> /tmp/app_dynamic
  Reason: tried: 'libmathutils.so' (no such file), ...
```

On Linux the equivalent fix is exporting `LD_LIBRARY_PATH` (or installing
the library to a system path like `/usr/lib` and running `ldconfig`); on
macOS it's `DYLD_LIBRARY_PATH`, or baking a search path into the binary
itself with `install_name_tool`/`-rpath` so it doesn't depend on an
environment variable at all. This dependency is exactly what breaks a
program with "works on my machine, fails on the deploy target" — the
executable is small and links instantly, but it's only actually complete
once its shared libraries are present wherever it runs.

**Tradeoff, and the mirror image of the static case**: one copy of
`libmathutils.so` on disk is shared (and shares physical memory) across
every process using it, and patching a bug means replacing that one file —
every program picks up the fix automatically next time it starts, with no
recompilation. The cost is exactly the fragility just demonstrated: the
library has to actually be present and findable at runtime, and an
incompatible version swapped in later can break a program that compiled
against a different one.

## Loading a shared library on purpose: `dlopen`/`dlsym`

Both examples above load the shared library automatically at process
startup. Sometimes you want to choose *at runtime* whether to load a
library at all (plugin systems, optional codecs, hot-swappable modules) —
`dlopen` does that explicitly:

```c
// dl_demo.c -- load a shared library at runtime with dlopen/dlsym
#include <stdio.h>
#include <dlfcn.h>

typedef int (*add_fn)(int, int);

int main(void) {
    void *handle = dlopen("./libmathutils.so", RTLD_LAZY);
    if (!handle) {
        fprintf(stderr, "dlopen failed: %s\n", dlerror());
        return 1;
    }

    dlerror();  // clear any existing error
    add_fn add = (add_fn)dlsym(handle, "add");
    char *err = dlerror();
    if (err != NULL) {
        fprintf(stderr, "dlsym failed: %s\n", err);
        dlclose(handle);
        return 1;
    }

    printf("dlopen'd add(10, 20) = %d\n", add(10, 20));

    dlclose(handle);
    return 0;
}
```

```bash
gcc -Wall -Wextra -o dl_demo dl_demo.c -ldl
./dl_demo
```

```text
dlopen'd add(10, 20) = 30
```

`dlsym` returns a `void *` that has to be cast to the correct function
pointer type (`add_fn` here) — there's no compiler-checked type safety at
this boundary, which is exactly why the code checks `dlerror()` explicitly
after the cast rather than trusting it. Get the function's signature wrong
and the cast still "succeeds"; calling the result is undefined behavior
that may not surface until the arguments happen to be interpreted wrong.

## Static vs. dynamic: cheat sheet

| | Static (`.a`) | Dynamic (`.so`/`.dylib`) |
|---|---|---|
| When code is linked | At compile/link time | At process startup (or via `dlopen`, on demand) |
| Copies of the code on disk | One per binary that links it | One, shared by every process |
| Executable size | Larger (includes the library) | Smaller (library is separate) |
| Runtime dependency | None — self-contained | Library file must be present and findable |
| Patch a bug in the library | Recompile and relink every dependent binary | Replace the one shared file |
| Build flags | `ar rcs`, then link with `-L -l` | `-fPIC` to compile, `-shared` to link the library |
| Inspect dependencies | `nm libname.a` | `ldd` (Linux) / `otool -L` (macOS) |

## Exercise

Add a third function, `int cube(int n)`, to `mathutils.c`/`mathutils.h`.
Rebuild **both** the static archive and the shared library, and rebuild
`app_static` and `app_dynamic` against the updated header without changing
`main.c`'s includes. Confirm both binaries print a correct `cube(4) = 64`.
Then, without recompiling `app_dynamic` again, replace `libmathutils.so`
with a version whose `add` function deliberately returns `a - b` instead of
`a + b`, and confirm `app_dynamic` immediately starts returning the wrong
answer the next time you run it — this is the dynamic-linking tradeoff from
the cheat sheet made concrete: the same binary, no recompilation, differs
in behavior purely because of what's sitting next to it on disk.
