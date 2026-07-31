# 03 · Multi-dimensional Arrays & String Manipulation

Grids, matrices, game boards, tables of records — the moment your data has two
axes you need a multi-dimensional array. And because C strings are just arrays
of `char`, "an array of strings" is a two-dimensional problem too, with a
choice of layouts that has real consequences.

This module covers how C actually lays 2-D arrays out in memory, how to pass
them to functions (the part that trips everyone up), and the string library
functions you'll use constantly — including which ones are dangerous and what
to use instead.

## A 2-D array is one flat block

`int grid[3][4]` is not "three pointers to rows". It is **twelve contiguous
ints**, stored row by row — C is *row-major*:

```c
#include <stdio.h>

int main(void) {
    int grid[3][4] = {
        {1,  2,  3,  4},
        {5,  6,  7,  8},
        {9, 10, 11, 12}
    };

    printf("sizeof(grid)    = %zu\n", sizeof grid);        // 12 ints
    printf("sizeof(grid[0]) = %zu\n", sizeof grid[0]);     // one row: 4 ints

    // Proof that it is one flat block:
    int *flat = &grid[0][0];
    printf("flat[6] = %d  (same as grid[1][2] = %d)\n", flat[6], grid[1][2]);

    for (int r = 0; r < 3; r++) {
        for (int c = 0; c < 4; c++) {
            printf("%3d", grid[r][c]);
        }
        printf("\n");
    }
    return 0;
}
// Output:
// sizeof(grid)    = 48
// sizeof(grid[0]) = 16
// flat[6] = 7  (same as grid[1][2] = 7)
//   1  2  3  4
//   5  6  7  8
//   9 10 11 12
```

`grid[r][c]` compiles to `*(&grid[0][0] + r * 4 + c)`. The compiler needs the
**column count** to do that multiplication — which is why the next section
exists.

Row-major order also has a performance consequence you'll meet again in
[Level 4](../level-4/03-performance-profiling.md): iterating rows-then-columns
walks memory sequentially and is cache-friendly; iterating columns-then-rows
jumps 16 bytes at a time and can be several times slower on large matrices.

## Passing a 2-D array to a function

A 2-D array decays to a pointer to its **first row**, i.e. `int (*)[4]`. The
column count is part of the type and cannot be omitted:

```c
#include <stdio.h>

#define COLS 4

// Any of these three parameter spellings mean the same thing:
//   int grid[3][COLS]   int grid[][COLS]   int (*grid)[COLS]
void print_grid(int rows, int grid[][COLS]) {
    for (int r = 0; r < rows; r++) {
        for (int c = 0; c < COLS; c++) printf("%3d", grid[r][c]);
        printf("\n");
    }
}

// Alternative: take it flat, and do the index maths yourself.
// This works for ANY dimensions decided at run time.
void print_flat(const int *data, int rows, int cols) {
    for (int r = 0; r < rows; r++) {
        for (int c = 0; c < cols; c++) printf("%3d", data[r * cols + c]);
        printf("\n");
    }
}

int main(void) {
    int grid[2][COLS] = {{1,2,3,4},{5,6,7,8}};
    print_grid(2, grid);
    printf("--\n");
    print_flat(&grid[0][0], 2, COLS);
    return 0;
}
// Output:
//   1  2  3  4
//   5  6  7  8
// --
//   1  2  3  4
//   5  6  7  8
```

Note `int (*grid)[COLS]` — pointer to an array of `COLS` ints. Drop the
parentheses and `int *grid[COLS]` means an *array of `COLS` pointers*, an
entirely different layout. This is the same parenthesis rule you saw with
function pointers in [Module 1](01-pointers-deep-dive.md).

## Two layouts for "array of strings"

```c
#include <stdio.h>

int main(void) {
    // (a) 2-D char array: fixed-width rows, all mutable, one block
    char names[3][16] = {"Ada", "Grace", "Linus"};
    names[0][0] = 'a';              // legal -- it's your own memory

    // (b) Array of pointers: ragged, each points at a string literal
    const char *words[] = {"short", "considerably-longer", "mid"};
    // words[0][0] = 's';           // UNDEFINED BEHAVIOR: literals are read-only

    for (int i = 0; i < 3; i++) printf("%s ", names[i]);
    printf("\n");
    for (int i = 0; i < 3; i++) printf("%s ", words[i]);
    printf("\n");

    printf("sizeof(names) = %zu, sizeof(words) = %zu\n",
           sizeof names, sizeof words);
    return 0;
}
// Output:
// ada Grace Linus
// short considerably-longer mid
// sizeof(names) = 48, sizeof(words) = 24
```

| | `char names[3][16]` | `const char *words[3]` |
|---|---------------------|------------------------|
| Memory | one 48-byte block | 3 pointers + the literals |
| Row length | all padded to 16 | exactly as long as needed |
| Mutable? | yes | no — literals are read-only |
| Swapping two entries | copies 16 bytes | swaps two pointers (cheap) |
| Good for | records you edit in place | fixed word lists, `argv`-style data |

Declaring string literals as `char *` instead of `const char *` compiles but
lets you write code that crashes at run time. Always `const`-qualify pointers
to literals.

## The string library

Every function here assumes a terminating `'\0'`. Forget it and they run off
the end of your buffer.

```c
#include <stdio.h>
#include <string.h>

int main(void) {
    char buf[32];

    strcpy(buf, "Hello");           // copy in
    strcat(buf, ", world");         // append
    printf("%s (len %zu)\n", buf, strlen(buf));

    printf("cmp equal:   %d\n", strcmp("abc", "abc"));   // 0
    printf("cmp a < b:   %d\n", strcmp("abc", "abd") < 0);

    char *found = strchr(buf, 'w');                       // first 'w'
    if (found) printf("found at index %td\n", found - buf);

    char *sub = strstr(buf, "world");                     // substring search
    if (sub) printf("substring: %s\n", sub);

    return 0;
}
// Output:
// Hello, world (len 12)
// cmp equal:   0
// cmp a < b:   1
// found at index 7
// substring: world
```

| Function | Does | Watch out for |
|----------|------|---------------|
| `strlen(s)` | length, excluding `'\0'` | O(n) — don't call it inside a loop condition |
| `strcpy(d, s)` | copy `s` into `d` | no bounds check — overflows silently |
| `strncpy(d, s, n)` | copy at most `n` bytes | may **not** terminate `d`; terminate manually |
| `strcat(d, s)` | append | walks `d` to find its end first; no bounds check |
| `strcmp(a, b)` | `<0`, `0`, `>0` | it is **not** a boolean — `if (strcmp(a,b))` means *not equal* |
| `strchr(s, c)` | pointer to first `c`, or `NULL` | check for `NULL` |
| `strstr(h, n)` | pointer to substring, or `NULL` | check for `NULL` |
| `snprintf(d, n, ...)` | formatted, always terminates | the safe general-purpose builder |

### The `strncpy` trap, and the fix

```c
#include <stdio.h>
#include <string.h>

int main(void) {
    char small[6];

    // strncpy does NOT add '\0' if the source fills the buffer exactly.
    strncpy(small, "abcdef", sizeof small);
    small[sizeof small - 1] = '\0';        // you must do this yourself
    printf("strncpy: %s\n", small);

    // snprintf always terminates, and tells you what it WANTED to write.
    char dest[10];
    int needed = snprintf(dest, sizeof dest, "%s-%d", "value", 12345);
    printf("snprintf: %s\n", dest);
    if (needed >= (int)sizeof dest) {
        printf("truncated: needed %d bytes\n", needed + 1);
    }
    return 0;
}
// Output:
// strncpy: abcde
// snprintf: value-123
// truncated: needed 12 bytes
```

`snprintf` is the workhorse: it never overflows, it always writes a `'\0'`, and
its return value tells you whether truncation happened. When in doubt, reach
for it instead of `strcpy`/`strcat`.

## Splitting a string with `strtok`

```c
#include <stdio.h>
#include <string.h>

int main(void) {
    // strtok MODIFIES its input, so it cannot be a string literal.
    char line[] = "widget,42,9.99";

    char *field = strtok(line, ",");
    while (field != NULL) {
        printf("[%s]\n", field);
        field = strtok(NULL, ",");   // NULL = "continue the same string"
    }

    printf("line is now: %s\n", line);   // shows the damage strtok did
    return 0;
}
// Output:
// [widget]
// [42]
// [9.99]
// line is now: widget
```

`strtok` replaces each delimiter with `'\0'` in place — which is why `line`
looks truncated afterwards, and why passing a literal (`char *line = "a,b";`)
is undefined behavior. It also keeps state in a hidden static variable, so you
cannot interleave two `strtok` loops or use it from threads.

## A worked example: word frequency in a fixed table

```c
#include <stdio.h>
#include <string.h>

#define MAX_WORDS 32
#define WORD_LEN  20

int main(void) {
    char text[] = "the cat sat on the mat the end";
    char words[MAX_WORDS][WORD_LEN];
    int  counts[MAX_WORDS] = {0};
    int  n = 0;

    for (char *tok = strtok(text, " "); tok != NULL; tok = strtok(NULL, " ")) {
        int found = -1;
        for (int i = 0; i < n; i++) {
            if (strcmp(words[i], tok) == 0) { found = i; break; }
        }
        if (found >= 0) {
            counts[found]++;
        } else if (n < MAX_WORDS) {
            snprintf(words[n], WORD_LEN, "%s", tok);   // safe bounded copy
            counts[n] = 1;
            n++;
        }
    }

    for (int i = 0; i < n; i++) printf("%-6s %d\n", words[i], counts[i]);
    return 0;
}
// Output:
// the    3
// cat    1
// sat    1
// on     1
// mat    1
// end    1
```

Note `strcmp(...) == 0` for "equal". Writing `if (strcmp(a, b))` reads like
"if equal" but actually means "if **different**" — a bug that survives code
review depressingly often.

## Exercise

Write a program that stores a 5×5 `int` matrix and implements three functions:
`void transpose(int m[][5], int n)` (swap in place), `int row_sum(const int
m[][5], int row)`, and `void print_matrix(const int m[][5], int n)`. Then add a
string half: given `char csv[] = "name,qty,price";`, use `strtok` to split it
and store the fields into a `char fields[8][16]` table with `snprintf`, printing
each field with its length. Confirm with `sizeof` that your matrix really is
100 contiguous bytes.
