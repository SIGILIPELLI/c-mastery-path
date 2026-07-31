# 07 · Modular Programming

[Level 1, Module 9](../level-1/09-preprocessor-multifile.md) showed the
mechanics: put declarations in a `.h`, definitions in a `.c`, compile both. This
module is about the *design* — how to decide what goes in a header, how to hide
implementation details behind `static`, what "linkage" actually means, and how
to build a module whose users cannot even see its internals.

A well-designed C module has one job, a small header describing what it does,
and a source file nobody else needs to read. Get this right and a 50,000-line
program stays workable; get it wrong and every change ripples through the whole
build.

## What belongs in a header

A header is a **contract**, not a dumping ground. It should contain only what
callers need to compile against your module.

| Put in the `.h` | Keep in the `.c` |
|-----------------|------------------|
| Public function **prototypes** | Function **bodies** |
| `typedef`s and structs callers must construct | Structs only you touch |
| `enum`s used in the public API | Internal constants |
| `#define`s that are part of the API | Implementation-only macros |
| Header guard | Everything `static` |

Two rules that save real pain:

- **Never define a variable or a non-inline function in a header.** Every `.c`
  that includes it gets its own copy, and the linker reports "duplicate symbol".
  Declare with `extern` in the header, define once in a `.c`.
- **Include what you use, and only what you use.** A header should include the
  headers *its own declarations* need (e.g. `<stddef.h>` if it mentions
  `size_t`) — and nothing more.

## `static` at file scope: the module's privacy keyword

Inside a function, `static` means "keep this variable between calls" — that's
the Level 1 meaning. At **file scope** it means something different and far more
important: *this name is not visible outside this translation unit*.

```c
// stats.c
#include "stats.h"
#include <stdlib.h>

// PRIVATE: no other .c file can call or even see these.
static int compare_doubles(const void *a, const void *b) {
    double x = *(const double *)a;
    double y = *(const double *)b;
    return (x > y) - (x < y);
}

static double sum_all(const double *v, size_t n) {
    double total = 0.0;
    for (size_t i = 0; i < n; i++) total += v[i];
    return total;
}

// PUBLIC: declared in stats.h
double stats_mean(const double *v, size_t n) {
    if (n == 0) return 0.0;
    return sum_all(v, n) / (double)n;
}
```

Marking helpers `static` gives you three concrete wins:

1. **No name collisions.** Another file can define its own `sum_all` and the
   linker won't care.
2. **Freedom to change.** Nothing outside `stats.c` can depend on it, so you can
   rename or delete it safely.
3. **Better optimization.** The compiler knows every call site, so it can inline
   aggressively or drop the function entirely if unused.

The habit: **make every function `static` by default**, and remove `static` only
when you deliberately publish it in the header.

## Linkage, in one table

"Linkage" is the rule that decides whether two declarations of the same name in
different files refer to the same thing.

| Declaration (at file scope) | Linkage | Meaning |
|------------------------------|---------|---------|
| `int counter;` | external | one global, shared across the program |
| `static int counter;` | internal | private to this `.c` file |
| `extern int counter;` | external (reference) | "defined elsewhere — link me to it" |
| `static void helper(void)` | internal | private function |
| `void helper(void)` | external | callable from any file |

### Sharing a global correctly

```c
// config.h
#ifndef CONFIG_H
#define CONFIG_H

extern int g_verbose;        // DECLARATION only -- no storage allocated
void config_set_verbose(int on);

#endif
```

```c
// config.c
#include "config.h"

int g_verbose = 0;           // THE definition -- exactly one, in one .c file

void config_set_verbose(int on) {
    g_verbose = on;
}
```

Write `int g_verbose = 0;` in the header instead and every file that includes it
defines its own — a duplicate-symbol link error, or worse, silently different
copies. (Globals are best avoided anyway; prefer passing state explicitly.)

## Opaque types: hiding the struct entirely

The strongest form of encapsulation in C. Callers get a pointer to a struct
whose *definition they never see*, so they cannot touch its fields — and
changing the layout doesn't force them to recompile.

**`counter.h`** — the public contract:

```c
// counter.h
#ifndef COUNTER_H
#define COUNTER_H

// Incomplete type: callers know Counter exists, not what's inside it.
typedef struct Counter Counter;

Counter *counter_create(const char *label);
void     counter_destroy(Counter *c);

void     counter_increment(Counter *c);
int      counter_value(const Counter *c);
const char *counter_label(const Counter *c);

#endif
```

**`counter.c`** — the private implementation:

```c
// counter.c
#include "counter.h"
#include <stdio.h>      // snprintf
#include <stdlib.h>

#define LABEL_MAX 32

// The full definition lives HERE and nowhere else.
struct Counter {
    char label[LABEL_MAX];
    int  value;
    int  increments;      // internal bookkeeping nobody outside knows about
};

Counter *counter_create(const char *label) {
    Counter *c = malloc(sizeof *c);
    if (c == NULL) return NULL;

    snprintf(c->label, LABEL_MAX, "%s", label ? label : "unnamed");
    c->value = 0;
    c->increments = 0;
    return c;
}

void counter_destroy(Counter *c) {
    free(c);                    // free(NULL) is safe, so no check needed
}

void counter_increment(Counter *c) {
    if (c == NULL) return;
    c->value++;
    c->increments++;
}

int counter_value(const Counter *c) {
    return c ? c->value : 0;
}

const char *counter_label(const Counter *c) {
    return c ? c->label : "";
}
```

**`main.c`** — the caller:

```c
// main.c
#include <stdio.h>
#include "counter.h"

int main(void) {
    Counter *hits = counter_create("page-hits");
    if (hits == NULL) return 1;

    for (int i = 0; i < 5; i++) counter_increment(hits);

    printf("%s = %d\n", counter_label(hits), counter_value(hits));

    // hits->value = 99;   // COMPILE ERROR: incomplete type, no members visible

    counter_destroy(hits);
    return 0;
}
// Output:
// page-hits = 5
```

```bash
gcc -Wall -Wextra -o app main.c counter.c
./app
```

This create/destroy pair is the standard C object idiom — `FILE *` works exactly
this way, which is why you've never seen inside a `FILE`. The rules of the
pattern:

- The header exposes a pointer type and functions; never the struct body.
- Every `_create` has exactly one matching `_destroy`, and the header documents
  who calls it.
- All functions take the object pointer first, and tolerate `NULL` gracefully.
- Prefix every public name with the module name (`counter_`), since C has no
  namespaces.

## Designing module boundaries

A few heuristics that hold up in real projects:

- **One responsibility per module.** If you can't name a module without "and",
  it's two modules.
- **Depend on headers, not on files.** If `a.c` needs something from `b.c`, it
  should include `b.h` — never declare `b`'s functions itself, or the two
  declarations will silently drift apart.
- **Avoid circular includes.** If `a.h` includes `b.h` and `b.h` includes `a.h`,
  the design is telling you the split is wrong. Forward-declare (`typedef struct
  Foo Foo;`) instead of including where you only need a pointer.
- **Keep headers cheap.** A header that pulls in ten others makes every build
  slower and every change more disruptive.

## Compiling and linking a multi-module program

```bash
# Compile each module to an object file independently
gcc -Wall -Wextra -c counter.c -o counter.o
gcc -Wall -Wextra -c stats.c   -o stats.o
gcc -Wall -Wextra -c main.c    -o main.o

# Link them into one executable
gcc counter.o stats.o main.o -o app
```

Change one `.c` and only that `.o` needs rebuilding. Change a `.h` and every
file that includes it must be rebuilt — which is precisely the dependency
tracking that [Module 8](08-makefiles-build-systems.md) automates with `make`.

Three linker and compiler errors you will meet, and what they mean:

| Error | Cause |
|-------|-------|
| `undefined reference to 'foo'` | You declared `foo` and called it, but never compiled/linked the `.c` that defines it |
| `duplicate symbol '_foo'` | `foo` is defined in two `.c` files (or defined in a header) |
| `implicit declaration of function 'foo'` | You called `foo` without including its header |

## Exercise

Build a three-file `Stack` module using the opaque-type pattern. `stack.h`
declares `typedef struct Stack Stack;` plus `stack_create(int capacity)`,
`stack_destroy`, `stack_push` (returning 1/0 for success as in
[Module 6](06-error-handling.md)), `stack_pop`, `stack_is_empty`, and
`stack_size`. `stack.c` defines the struct with a heap-allocated `int *data`
array (see [Module 2](02-dynamic-memory.md)) and keeps at least one `static`
helper such as `static int stack_is_full(const Stack *s)`. `main.c` pushes ten
values, pops them all, and prints them — proving they come out in reverse order.
Then try to write `s->data[0] = 1;` in `main.c` and confirm the compiler refuses.
