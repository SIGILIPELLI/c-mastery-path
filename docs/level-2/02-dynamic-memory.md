# 02 · Dynamic Memory

Everything you've allocated so far has had a size fixed at compile time:
`int scores[100];` reserves room for exactly 100 integers, whether you need 3
or 3000. **Dynamic memory** lifts that restriction. With `malloc` and friends
you ask the operating system for memory at *run time*, sized by whatever your
program has just discovered it needs.

The cost is responsibility. C has no garbage collector. Every block you
allocate is yours until you `free` it, and the two most expensive bug classes
in the language — memory leaks and use-after-free — live entirely in this
module.

## Stack vs heap

| | Stack | Heap |
|---|-------|------|
| Allocated by | declaring a local variable | `malloc` / `calloc` / `realloc` |
| Freed by | automatically, when the function returns | you, via `free` |
| Size | fixed at compile time, small (often 1–8 MB total) | limited by available RAM |
| Speed | very fast (just moves a pointer) | slower (bookkeeping, may hit the OS) |
| Lifetime | ends with the enclosing block | until you free it |

The lifetime row is the important one. A local array dies when its function
returns; heap memory survives, which is what lets a function build a data
structure and hand it back to its caller.

## `malloc` — allocate raw bytes

`malloc(n)` returns a pointer to `n` **bytes** of uninitialized memory, or
`NULL` if the request cannot be satisfied.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int n;
    printf("How many numbers? ");
    if (scanf("%d", &n) != 1 || n <= 0) {
        printf("Invalid count.\n");
        return 1;
    }

    // Size in BYTES: n elements x the size of one element.
    int *nums = malloc(n * sizeof *nums);
    if (nums == NULL) {                  // ALWAYS check
        printf("Out of memory.\n");
        return 1;
    }

    for (int i = 0; i < n; i++) {
        nums[i] = (i + 1) * (i + 1);     // heap memory indexes just like an array
    }

    for (int i = 0; i < n; i++) {
        printf("%d ", nums[i]);
    }
    printf("\n");

    free(nums);        // hand it back
    nums = NULL;       // and make the dangling pointer harmless
    return 0;
}
// Input:  5
// Output: 1 4 9 16 25
```

Three habits in that snippet, all worth copying:

1. **`sizeof *nums`, not `sizeof(int)`.** Write the size in terms of the
   pointer, and the line stays correct if the type ever changes to `long`.
2. **Check for `NULL`.** `malloc` can fail. Dereferencing the result without
   checking turns a recoverable condition into a crash.
3. **Set the pointer to `NULL` after freeing.** `free(NULL)` is a no-op, so a
   nulled pointer is safe to free again; a stale non-null pointer is not.

Note that `malloc` does **not** zero the memory. Reading `nums[i]` before
writing to it reads whatever bytes the allocator happened to hand you —
undefined behavior, and it will not reliably be zero.

## `calloc` — allocate and zero

`calloc(count, size)` takes two arguments, multiplies them (with an internal
overflow check), and zeroes the result:

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    size_t n = 5;
    int *counts = calloc(n, sizeof *counts);   // all elements start at 0
    if (counts == NULL) return 1;

    for (size_t i = 0; i < n; i++) {
        printf("%d ", counts[i]);
    }
    printf("\n");

    free(counts);
    return 0;
}
// Output:
// 0 0 0 0 0
```

Use `calloc` when zero is a meaningful starting value (counters, histograms,
flags). Use `malloc` when you're about to overwrite every byte anyway — zeroing
is not free.

## `realloc` — grow or shrink an existing block

`realloc(ptr, newsize)` returns a block of `newsize` bytes containing the old
contents (truncated if smaller). It may return the *same* address or move the
data to a new one.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    size_t capacity = 2;
    size_t count = 0;
    int *items = malloc(capacity * sizeof *items);
    if (items == NULL) return 1;

    for (int value = 1; value <= 5; value++) {
        if (count == capacity) {
            size_t new_cap = capacity * 2;         // grow geometrically

            // NEVER assign realloc's result straight back to "items"
            int *tmp = realloc(items, new_cap * sizeof *items);
            if (tmp == NULL) {
                free(items);                       // original is still valid
                printf("Grow failed.\n");
                return 1;
            }
            items = tmp;
            capacity = new_cap;
            printf("(grew to capacity %zu)\n", capacity);
        }
        items[count++] = value * 10;
    }

    for (size_t i = 0; i < count; i++) printf("%d ", items[i]);
    printf("\n");

    free(items);
    return 0;
}
// Output:
// (grew to capacity 4)
// (grew to capacity 8)
// 10 20 30 40 50
```

The `tmp` variable is not stylistic fussiness. If you write
`items = realloc(items, ...)` and the call fails, `realloc` returns `NULL`
while the original block is still allocated — you've just overwritten the only
pointer to it and leaked it permanently.

Doubling the capacity rather than adding one keeps the total work linear.
Growing by one each time makes appending *n* items cost O(n²) copies.

## The four ways this goes wrong

```c
#include <stdlib.h>
#include <string.h>

void bugs(void) {
    // 1. LEAK -- allocated, never freed
    int *a = malloc(100 * sizeof *a);
    (void)a;   // ... function returns, address lost forever

    // 2. USE AFTER FREE -- reading memory you gave back
    int *b = malloc(sizeof *b);
    free(b);
    // *b = 5;                 // undefined behavior

    // 3. DOUBLE FREE -- freeing the same block twice
    int *c = malloc(sizeof *c);
    free(c);
    // free(c);                // undefined behavior; often corrupts the heap

    // 4. BUFFER OVERFLOW -- writing past the end
    char *d = malloc(5);
    // strcpy(d, "hello");     // "hello" needs 6 bytes: 5 chars + '\0'
    free(d);
}
```

None of these reliably crash on the spot. A leak just makes the program grow;
a use-after-free may read plausible-looking garbage for months before it
corrupts something important. That's why you use tools rather than eyeballs:
[Module 9](09-debugging-tools.md) covers running your program under Valgrind
and AddressSanitizer, which catch all four of these automatically.

## Returning heap memory from a function

This is the pattern that makes dynamic memory worth the trouble — a function
producing a result whose size it decides:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// Returns a NEW heap string that the caller must free().
char *join_strings(const char *a, const char *b, const char *sep) {
    size_t len = strlen(a) + strlen(sep) + strlen(b) + 1;   // +1 for '\0'

    char *out = malloc(len);
    if (out == NULL) return NULL;

    strcpy(out, a);
    strcat(out, sep);
    strcat(out, b);
    return out;
}

int main(void) {
    char *s = join_strings("dynamic", "memory", " ");
    if (s == NULL) return 1;

    printf("%s (%zu chars)\n", s, strlen(s));

    free(s);      // caller's job -- documented in the comment above
    return 0;
}
// Output:
// dynamic memory (14 chars)
```

The comment `caller must free()` is doing real work. C has no way to express
ownership in the type system, so **ownership lives in your documentation**.
Every function that returns heap memory should say so, and every project should
be consistent about who frees what.

Contrast this with the broken version:

```c
char *broken(void) {
    char buf[64];          // lives on the stack
    strcpy(buf, "hello");
    return buf;            // BUG: buf dies when broken() returns
}
```

Returning the address of a local is one of C's most reliable ways to produce a
crash that only shows up in production.

## Cheat sheet

| Call | Returns | Initialized? | Notes |
|------|---------|--------------|-------|
| `malloc(n)` | `n` bytes, or `NULL` | no — garbage | fastest; you must fill it |
| `calloc(c, n)` | `c * n` bytes, or `NULL` | yes — all zero bytes | checks `c * n` for overflow |
| `realloc(p, n)` | `n` bytes, or `NULL` | old contents preserved | may move the block; use a temp |
| `free(p)` | nothing | — | `free(NULL)` is safe and does nothing |

## Exercise

Write `int *read_numbers(size_t *out_count)` that reads integers from `stdin`
until EOF (`scanf("%d", &x) != 1`) into a heap array that starts with capacity 4
and doubles with `realloc` whenever it fills up. Store the final count through
`out_count` and return the array. In `main`, call it, print the values and their
average, then `free` the array exactly once. Test it by piping input:
`echo "3 1 4 1 5 9 2 6" | ./a.out`. Then re-read
[Module 1](01-pointers-deep-dive.md) on why the function needs `size_t *` rather
than `size_t` for the count.
