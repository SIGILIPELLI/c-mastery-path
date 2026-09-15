---
description: "Pointers Deep Dive — Level 1, Module 6 introduced pointers as variables that hold addresses. That's the what. This module is the how far — pointer…"
---

# 01 · Pointers Deep Dive

[Level 1, Module 6](../level-1/06-pointers-basics.md) introduced pointers as
variables that hold addresses. That's the *what*. This module is the *how far* —
pointer arithmetic, the exact relationship between arrays and pointers, `const`
correctness, `void *`, and function pointers.

These are the tools that make C's standard library work the way it does.
`qsort` sorts anything because it takes a `void *` array and a function pointer.
`strlen` walks a string by incrementing a pointer. Once pointer arithmetic
clicks, a large amount of "weird" C code stops being weird.

## Pointer arithmetic is scaled by type

Adding `1` to a pointer does **not** add one byte. It advances the pointer by
`sizeof(*ptr)` bytes — one whole element:

```c
#include <stdio.h>

int main(void) {
    int   nums[]    = {10, 20, 30, 40};
    char  letters[] = {'a', 'b', 'c', 'd'};

    int  *ip = nums;
    char *cp = letters;

    printf("ip     = %p\n", (void *)ip);
    printf("ip + 1 = %p   (moved %zu bytes)\n",
           (void *)(ip + 1), sizeof(int));

    printf("cp     = %p\n", (void *)cp);
    printf("cp + 1 = %p   (moved %zu byte)\n",
           (void *)(cp + 1), sizeof(char));

    return 0;
}
// Output (addresses vary):
// ip     = 0x7ffee0a1c9a0
// ip + 1 = 0x7ffee0a1c9a4   (moved 4 bytes)
// cp     = 0x7ffee0a1c990
// cp + 1 = 0x7ffee0a1c991   (moved 1 byte)
```

The compiler knows the pointed-to type, so it multiplies your `+1` by the size
of that type. This is exactly why `nums[i]` and `*(nums + i)` are equivalent:
indexing *is* scaled pointer arithmetic with a friendlier syntax.

| Expression | Meaning |
|------------|---------|
| `p + n` | Address `n` **elements** forward (not `n` bytes) |
| `p - n` | Address `n` elements backward |
| `p2 - p1` | Number of **elements** between two pointers into the same array |
| `*p++` | Read `*p`, then advance `p` (postfix `++` binds tighter than `*`) |
| `(*p)++` | Increment the **value** `p` points at |
| `p[i]` | Exactly `*(p + i)` |

### Walking an array with a pointer

```c
#include <stdio.h>

int main(void) {
    int nums[] = {3, 1, 4, 1, 5, 9};
    int n = sizeof(nums) / sizeof(nums[0]);

    int *begin = nums;
    int *end   = nums + n;   // one PAST the last element -- legal to form

    int sum = 0;
    for (int *p = begin; p < end; p++) {
        sum += *p;
    }

    printf("Elements: %d\n", (int)(end - begin));   // pointer difference
    printf("Sum:      %d\n", sum);

    return 0;
}
// Output:
// Elements: 6
// Sum:      23
```

Two rules worth memorizing:

- Forming a pointer **one past the end** of an array is legal (that's what
  makes `p < end` loops work), but **dereferencing** it is undefined behavior.
- Subtracting two pointers is only defined when both point into the *same*
  array. Comparing pointers from unrelated allocations is undefined too, even
  though it usually "works".

## Arrays are not pointers

They *decay* into pointers in most expressions, but they are not the same type.
The difference shows up the moment you take `sizeof`:

```c
#include <stdio.h>

void by_param(int arr[]) {          // silently rewritten to: int *arr
    printf("inside function: sizeof(arr) = %zu\n", sizeof(arr));
}

int main(void) {
    int nums[10];
    printf("in main: sizeof(nums) = %zu\n", sizeof(nums));
    by_param(nums);
    return 0;
}
// Output on a 64-bit machine:
// in main: sizeof(nums) = 40         (10 ints x 4 bytes)
// inside function: sizeof(arr) = 8   (just a pointer!)
```

This is the single most common beginner trap in C: **array size information is
lost when you pass an array to a function.** `int arr[]` in a parameter list is
a lie — the compiler turns it into `int *arr`. That's why nearly every C
function that takes an array also takes a length:

```c
int sum_array(const int *arr, size_t n);   // honest signature
```

## `const` and pointers: where you put it matters

`const` placement changes *what* is constant. Read the declaration from the
variable name outward:

```c
#include <stdio.h>

int main(void) {
    int a = 1, b = 2;

    const int *p1 = &a;     // pointer to const int: *p1 is read-only
    // *p1 = 5;             // ERROR: cannot assign to *p1
    p1 = &b;                // fine: the pointer itself can move

    int *const p2 = &a;     // const pointer to int: p2 cannot move
    *p2 = 5;                // fine: the value can change
    // p2 = &b;             // ERROR: cannot reassign p2

    const int *const p3 = &a;   // neither can change

    printf("a = %d, *p1 = %d, *p3 = %d\n", a, *p1, *p3);
    return 0;
}
// Output:
// a = 5, *p1 = 2, *p3 = 5
```

| Declaration | Can `p` point elsewhere? | Can you write through `*p`? |
|-------------|--------------------------|------------------------------|
| `int *p` | yes | yes |
| `const int *p` | yes | no |
| `int *const p` | no | yes |
| `const int *const p` | no | no |

Marking a parameter `const int *` documents "I will only read this" and lets
the compiler catch accidental writes. Get in the habit — it's free safety.

## `void *` — the generic pointer

`void *` can hold the address of *any* object type. It is how `malloc`,
`memcpy`, and `qsort` stay type-agnostic:

```c
#include <stdio.h>
#include <stddef.h>

// Swap any two objects of the same size, byte by byte.
void generic_swap(void *a, void *b, size_t size) {
    unsigned char *pa = a;
    unsigned char *pb = b;
    for (size_t i = 0; i < size; i++) {
        unsigned char tmp = pa[i];
        pa[i] = pb[i];
        pb[i] = tmp;
    }
}

int main(void) {
    int x = 1, y = 2;
    generic_swap(&x, &y, sizeof x);
    printf("x=%d y=%d\n", x, y);

    double d1 = 1.5, d2 = 9.75;
    generic_swap(&d1, &d2, sizeof d1);
    printf("d1=%.2f d2=%.2f\n", d1, d2);

    return 0;
}
// Output:
// x=2 y=1
// d1=9.75 d2=1.50
```

Two rules for `void *`:

- You **cannot dereference** a `void *` — there's no type, so no size to read.
  Convert to a concrete pointer type first (as `unsigned char *` above).
- In C (unlike C++), conversion to and from `void *` is implicit. Casting the
  result of `malloc` is unnecessary and can hide a missing `#include`.

## Function pointers

A function name also decays to a pointer — to code instead of data. A function
pointer lets you store *which* function to call in a variable:

```c
#include <stdio.h>

int add(int a, int b)      { return a + b; }
int subtract(int a, int b) { return a - b; }
int multiply(int a, int b) { return a * b; }

int main(void) {
    // "op" is a pointer to a function taking (int, int) and returning int
    int (*op)(int, int);

    op = add;                       // &add works too; the & is optional
    printf("add:      %d\n", op(4, 2));

    op = subtract;
    printf("subtract: %d\n", op(4, 2));

    // A table of operations -- dispatch without a switch statement
    int (*ops[])(int, int) = { add, subtract, multiply };
    const char *names[] = { "+", "-", "*" };

    for (int i = 0; i < 3; i++) {
        printf("4 %s 2 = %d\n", names[i], ops[i](4, 2));
    }

    return 0;
}
// Output:
// add:      6
// subtract: 2
// 4 + 2 = 6
// 4 - 2 = 2
// 4 * 2 = 8
```

The parentheses in `int (*op)(int, int)` matter. Without them,
`int *op(int, int)` declares a *function returning `int *`* — a completely
different thing. A `typedef` makes these readable:

```c
typedef int (*BinaryOp)(int, int);

BinaryOp op = multiply;      // much easier to read
```

### The classic use: `qsort`

```c
#include <stdio.h>
#include <stdlib.h>

// Comparator: return <0, 0, or >0. qsort hands us void pointers to elements.
int compare_ints(const void *a, const void *b) {
    int x = *(const int *)a;
    int y = *(const int *)b;
    return (x > y) - (x < y);   // avoids the overflow that "x - y" can cause
}

int main(void) {
    int nums[] = {42, 7, 19, 3, 88, 1};
    size_t n = sizeof(nums) / sizeof(nums[0]);

    qsort(nums, n, sizeof(nums[0]), compare_ints);

    for (size_t i = 0; i < n; i++) {
        printf("%d ", nums[i]);
    }
    printf("\n");
    return 0;
}
// Output:
// 1 3 7 19 42 88
```

Note the comparator returns `(x > y) - (x < y)` rather than `x - y`. For large
values `x - y` can overflow `int` — signed overflow is undefined behavior in C,
and the sort can silently produce wrong results.

## Pointer-to-pointer, used for real

`**` matters when a function must change *where the caller's pointer points*:

```c
#include <stdio.h>
#include <stdlib.h>

// Takes the ADDRESS of the caller's pointer so it can reassign it.
void allocate_row(int **out, int n) {
    *out = malloc(n * sizeof(int));
    if (*out == NULL) return;
    for (int i = 0; i < n; i++) {
        (*out)[i] = i * i;      // parentheses required: *out[i] means something else
    }
}

int main(void) {
    int *row = NULL;
    allocate_row(&row, 5);

    if (row != NULL) {
        for (int i = 0; i < 5; i++) printf("%d ", row[i]);
        printf("\n");
        free(row);
    }
    return 0;
}
// Output:
// 0 1 4 9 16
```

If `allocate_row` took `int *out` instead, it would receive a *copy* of the
caller's `NULL` pointer, and the caller's `row` would still be `NULL` on
return. Dynamic allocation is covered fully in
[Module 2](02-dynamic-memory.md).

## How It Actually Works

The `int (*op)(int, int)` syntax reflects something real about how functions
exist in a compiled binary: a function is just a label marking the address
of its first instruction inside the executable's code segment. `op = add;`
copies that address — the same 8-byte value on a 64-bit machine that a data
pointer would hold — into `op`'s storage. Calling `op(4, 2)` compiles to
"load the address out of `op`, then issue an indirect call instruction to
that address" (`call *%rax` in x86-64 assembly) instead of a direct
`call add` whose target is baked into the instruction at compile time. This
is the mechanical difference between calling a function normally and
calling one "by pointer": the CPU doesn't know or care which function it's
about to jump to until the moment the indirect call executes, which is
exactly what lets `qsort` invoke a comparator it has never heard of at
compile time — the comparator's address is simply data qsort was handed at
runtime.

This is also why `void *` cannot be dereferenced directly: dereferencing
requires knowing how many bytes to read and how to interpret the bit
pattern (an `int` needs a 4-byte two's-complement load; a `double` needs an
8-byte IEEE-754 load into a floating-point register) — information a
`void *` deliberately discards. `qsort`'s comparator receiving `const void
*a` is handed a raw address with zero type information; casting to
`const int *` before dereferencing is what tells the compiler "interpret
the next 4 bytes at this address as an int," restoring the information
`void *` erased.

`const` qualifiers, in contrast, are a pure compile-time fiction with zero
runtime representation: `const int *p1` and `int *p1` produce byte-for-byte
identical machine code for reading `*p1` — the compiler simply refuses to
*generate* code that would write through `p1`, catching the mistake before
code exists at all. There is no protected-memory mechanism backing this (the
OS-level read-only protection you get from `const` global data placed in a
read-only segment is a separate, real mechanism); a `const int *` aimed at
genuinely mutable stack memory can still be defeated by an explicit cast,
which is exactly why `const` is described as a compiler discipline tool, not
a security boundary.

The "array decays to pointer, losing size information" behavior demonstrated
by `by_param` is a direct consequence of the calling convention: a function
parameter is just a register or stack slot sized for one pointer (8 bytes),
so there is physically nowhere for the compiler to also pass "and by the
way, this points into a block of 10 elements" — that metadata simply isn't
part of the ABI (Application Binary Interface) that describes how arguments
are passed. Passing the length as an explicit second parameter is the only
way to recover information the calling convention itself throws away.

## Exercise

Write a program with a function
`int *find_first(int *arr, size_t n, int (*pred)(int));` that returns a pointer
to the first element satisfying `pred`, or `NULL` if none match. Implement two
predicates, `is_even` and `is_negative`, and walk the array using pointer
arithmetic only — no `[]` indexing inside `find_first`. In `main`, call it with
each predicate, and when a match is found print both the value and its index,
computed as a pointer difference (`result - arr`).
