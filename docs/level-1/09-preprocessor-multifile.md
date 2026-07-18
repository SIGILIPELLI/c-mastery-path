# 09 · Preprocessor & Multi-file Compilation

Before a single line of C is actually compiled, a separate pass called the
**preprocessor** runs over your source, handling every line that starts with
`#`. It textually includes files, substitutes constants, and expands macros —
all before the compiler ever sees "real" C syntax. Understanding the
preprocessor is also the key to splitting a program across multiple files,
which is how every non-trivial C project is organized.

## `#include`

`#include` tells the preprocessor to paste the contents of another file in
right where the directive appears:

```c
#include <stdio.h>   // angle brackets: search the system/standard library paths
#include "helpers.h" // quotes: search relative to the current file first
```

- **Angle brackets** (`<stdio.h>`) are for standard library or system headers.
- **Quotes** (`"helpers.h"`) are for your own project headers — the
  preprocessor looks in the same directory as the including file first, then
  falls back to the system paths.

## `#define` — constants and simple macros

`#define` creates a text substitution that the preprocessor performs
everywhere the name appears:

```c
#include <stdio.h>

#define MAX_STUDENTS 30
#define PI 3.14159

// A simple function-like macro
#define SQUARE(x) ((x) * (x))

int main(void) {
    printf("Max students: %d\n", MAX_STUDENTS);
    printf("Circle area with r=2: %.2f\n", PI * SQUARE(2));

    return 0;
}
// Output:
// Max students: 30
// Circle area with r=2: 12.57
```

Notice the extra parentheses in `((x) * (x))` — the preprocessor does *pure
text substitution*, so `SQUARE(x)` without those parentheses would expand
`SQUARE(1 + 2)` into `1 + 2 * 1 + 2` (wrong) instead of `((1 + 2) * (1 + 2))`
(right). This is a classic macro pitfall; for anything beyond a trivial
constant, a real function is usually clearer and safer than a macro.

## Header guards: `#ifndef` / `#define` / `#endif`

If the same header ends up `#include`d more than once in a single compiled
file (easy to happen once headers include other headers), the compiler sees
the same declarations twice and fails with "redefinition" errors. A **header
guard** prevents this by making the second inclusion a no-op:

```c
// math_utils.h
#ifndef MATH_UTILS_H
#define MATH_UTILS_H

int add(int a, int b);
int multiply(int a, int b);

#endif
```

The first time this file is included, `MATH_UTILS_H` isn't defined yet, so the
preprocessor defines it and includes everything between `#ifndef` and
`#endif`. Any later `#include "math_utils.h"` in the same compilation sees
`MATH_UTILS_H` already defined and skips the body entirely. Every header you
write should have a guard, using a name unlikely to collide with anything
else (commonly the filename in uppercase, as above).

## Splitting a program across files

A header (`.h`) declares *what* exists (function signatures, constants); one
or more source files (`.c`) define *how* it works. This separation lets
`main.c` use functions it doesn't itself contain the code for.

**`math_utils.h`** — declares what's available:

```c
// math_utils.h
#ifndef MATH_UTILS_H
#define MATH_UTILS_H

int add(int a, int b);
int multiply(int a, int b);

#endif
```

**`math_utils.c`** — implements it:

```c
// math_utils.c
#include "math_utils.h"

int add(int a, int b) {
    return a + b;
}

int multiply(int a, int b) {
    return a * b;
}
```

**`main.c`** — uses it:

```c
// main.c
#include <stdio.h>
#include "math_utils.h"

int main(void) {
    int sum = add(3, 4);
    int product = multiply(3, 4);

    printf("Sum: %d\n", sum);
    printf("Product: %d\n", product);

    return 0;
}
// Output:
// Sum: 7
// Product: 12
```

`main.c` never sees the *implementation* of `add` and `multiply` — only the
declarations from `math_utils.h`. That's enough for the compiler to check the
calls are used correctly; the actual code gets tied together at the linking
step.

## Compiling multiple files with `gcc`

To build a program made of several `.c` files, list all of them on the `gcc`
command line:

```bash
gcc main.c math_utils.c -o program
./program
# Sum: 7
# Product: 12
```

`gcc` compiles each `.c` file and then **links** them together into a single
executable named `program`. `math_utils.h` doesn't need to be listed — it's
pulled in automatically wherever a `.c` file `#include`s it.

## A preview of separate compilation

For a two-file project like this, compiling everything at once with a single
`gcc` command is fine. But recompiling *every* file every time you change
*one line* in a large project wastes a lot of time. `gcc -c` compiles a single
file into an **object file** (`.o`) — machine code that isn't yet linked into
a full program — without needing the rest of the project:

```bash
gcc -c math_utils.c -o math_utils.o   # compile only
gcc -c main.c -o main.o               # compile only
gcc main.o math_utils.o -o program    # link the object files together
```

If you only change `main.c`, you only need to recompile `main.o` and re-link —
`math_utils.o` doesn't need to be touched. Automating this file-by-file
recompilation is exactly what **Makefiles** are for, covered in
[Level 2, Module 8](../level-2/08-makefiles-build-systems.md).

| Directive / concept | Purpose |
|----------------------|---------|
| `#include <...>` | Include a standard/system header |
| `#include "..."` | Include a project header (searched locally first) |
| `#define NAME value` | Define a constant or simple macro |
| `#ifndef` / `#define` / `#endif` | Header guard — prevent duplicate inclusion |
| `.h` file | Declarations shared across `.c` files |
| `gcc a.c b.c -o prog` | Compile and link multiple source files at once |
| `gcc -c a.c -o a.o` | Compile one file to an object file, without linking |

## Exercise

Create a small two-file library: a header `strings_utils.h` with a header
guard declaring a function `int countVowels(const char *text);`, and a source
file `strings_utils.c` implementing it (loop over the string, as in
[Module 5](05-arrays-strings.md), counting `a/e/i/o/u`, both cases). Write a
`main.c` that includes your header, calls `countVowels` on a couple of test
strings, and prints the results. Compile it with a single `gcc` command that
lists both `.c` files.
