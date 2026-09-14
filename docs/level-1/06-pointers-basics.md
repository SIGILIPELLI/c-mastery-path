# 06 · Pointers Basics

## 🎥 Video walkthrough

<iframe width="100%" height="400" style="max-width:720px;aspect-ratio:16/9;height:auto;" src="https://www.youtube.com/embed/gNlQXuHazmU" title="Video walkthrough" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

Every variable lives somewhere in memory. A pointer is just a variable whose
value *is* a memory address — instead of holding a number or a character, it
holds the location where a number or character lives. Pointers are the
foundation for arrays, strings, dynamic memory, and passing data efficiently
between functions in C.

This module only covers the basics — declaring, reading, and using pointers
safely. Pointer arithmetic and function pointers get a full deep dive in
[Level 2, Module 1](../level-2/01-pointers-deep-dive.md).

## The address-of operator (`&`)

Every variable has an address. The `&` operator gives you that address:

```c
#include <stdio.h>

int main(void) {
    int age = 30;

    printf("Value of age:   %d\n", age);
    printf("Address of age: %p\n", (void *)&age);

    return 0;
}
// Output (address will vary each run):
// Value of age:   30
// Address of age: 0x7ffee3a1c9ac
```

`%p` is the format specifier for printing addresses; casting to `(void *)` is
the conventional, portable way to pass a pointer to `printf`.

## Declaring a pointer and the dereference operator (`*`)

A pointer variable is declared with a type and a `*`, and it must be told
*what type of thing* it points to:

```c
#include <stdio.h>

int main(void) {
    int age = 30;
    int *agePtr = &age;   // agePtr holds the address of age

    printf("agePtr holds address: %p\n", (void *)agePtr);
    printf("Value at that address (dereferenced): %d\n", *agePtr);

    *agePtr = 31;   // changes age itself, through the pointer
    printf("age is now: %d\n", age);

    return 0;
}
// Output:
// agePtr holds address: 0x7ffee3a1c9ac
// Value at that address (dereferenced): 30
// age is now: 31
```

Two very different meanings for `*` show up here:

- In a **declaration** (`int *agePtr`), `*` says "this variable is a pointer."
- In an **expression** (`*agePtr = 31`), `*` means "dereference — go to the
  address this pointer holds, and read/write the value there."

| Operator | Name | Meaning |
|----------|------|---------|
| `&x` | Address-of | "Give me the memory address of `x`" |
| `*p` | Dereference | "Give me the value stored at the address `p` holds" |
| `int *p` | Pointer declaration | "`p` is a pointer to an `int`" |

## NULL pointers

An uninitialized pointer holds a garbage address — dereferencing it is
undefined behavior and a common source of crashes. It's good practice to
initialize a pointer to `NULL` when it doesn't yet point anywhere, and to
check before dereferencing:

```c
#include <stdio.h>
#include <stddef.h>   // defines NULL

int main(void) {
    int *ptr = NULL;

    if (ptr == NULL) {
        printf("ptr does not point to anything yet.\n");
    }

    int value = 42;
    ptr = &value;

    if (ptr != NULL) {
        printf("ptr now points to a value: %d\n", *ptr);
    }

    return 0;
}
// Output:
// ptr does not point to anything yet.
// ptr now points to a value: 42
```

Checking for `NULL` before dereferencing is one of the most important habits
in C — functions like `malloc` (covered in Level 2) return `NULL` on failure,
and dereferencing that `NULL` without checking crashes the program.

## Pointers and arrays

An array name, when used in most expressions, "decays" into a pointer to its
first element. This is why arrays and pointers feel closely related in C:

```c
#include <stdio.h>

int main(void) {
    int numbers[] = {10, 20, 30, 40};

    printf("numbers itself:      %p\n", (void *)numbers);
    printf("&numbers[0]:         %p\n", (void *)&numbers[0]);
    printf("First element:       %d\n", *numbers);        // same as numbers[0]
    printf("Second via pointer:  %d\n", *(numbers + 1));  // same as numbers[1]

    return 0;
}
// Output:
// numbers itself:      0x7ffee3a1c990
// &numbers[0]:         0x7ffee3a1c990    (identical address)
// First element:       10
// Second via pointer:  20
```

`numbers` and `&numbers[0]` print the same address — the array name decays to
a pointer to its first element. `numbers[i]` and `*(numbers + i)` are
equivalent ways of writing the same access. The full rules of pointer
arithmetic (why `+1` moves by `sizeof(int)` bytes, not one byte) are covered in
[Level 2, Module 1](../level-2/01-pointers-deep-dive.md); for now, just
recognize that arrays and pointers are closely related.

## Pass-by-reference with pointers

C passes arguments to functions **by value** — a function normally gets a
*copy* of the argument and can't modify the caller's variable. Pointers let a
function reach back and modify the original, simulating pass-by-reference:

```c
#include <stdio.h>

// Without a pointer, this would only swap the local copies
void swap(int *a, int *b) {
    int temp = *a;
    *a = *b;
    *b = temp;
}

int main(void) {
    int x = 5;
    int y = 10;

    printf("Before swap: x=%d, y=%d\n", x, y);
    swap(&x, &y);   // pass addresses, not values
    printf("After swap:  x=%d, y=%d\n", x, y);

    return 0;
}
// Output:
// Before swap: x=5, y=10
// After swap:  x=10, y=5
```

Without pointers, `swap` would receive copies of `x` and `y`; changes inside
the function would vanish when it returned. By passing `&x` and `&y`, `swap`
receives the *addresses*, dereferences them, and modifies the caller's actual
variables.

## Pointer to pointer (brief look)

A pointer can itself be pointed to, using `**`. This shows up when a function
needs to modify a pointer that lives in the caller (for example, allocating
memory and handing the new address back):

```c
#include <stdio.h>

int main(void) {
    int value = 100;
    int *ptr = &value;      // ptr points to value
    int **ptrToPtr = &ptr;  // ptrToPtr points to ptr

    printf("value:            %d\n", value);
    printf("*ptr:             %d\n", *ptr);
    printf("**ptrToPtr:       %d\n", **ptrToPtr);

    return 0;
}
// Output:
// value:            100
// *ptr:             100
// **ptrToPtr:       100
```

This is just a preview — you'll use pointer-to-pointer patterns more
deliberately once dynamic memory allocation is introduced in
[Level 2](../level-2/02-dynamic-memory.md).

## How It Actually Works

A pointer variable itself is not magic — it's an ordinary chunk of memory
that happens to store a number: the address of another byte in the
process's address space. `int *agePtr = &age;` allocates 8 bytes on a
64-bit machine (a pointer's size equals the machine's address width, not
the size of what it points to) and stores `age`'s address in them. `&age`
is computed by the compiler at compile time if `age` is a known stack
offset — it's literally "current frame base plus this variable's fixed
offset," not a runtime lookup.

Dereferencing (`*agePtr`) compiles into a two-step memory access at the
hardware level: first load the address value out of `agePtr`'s own storage,
then issue a second load/store using *that* value as the address. This is
exactly why an uninitialized pointer is dangerous — its bytes hold whatever
garbage was left on the stack from a previous function call, and
dereferencing it means asking the CPU to read/write at a essentially random
address, which either corrupts unrelated memory silently or hits a page the
OS hasn't mapped into your process, triggering a **segmentation fault**
(the kernel's memory-management unit rejects the access and sends
`SIGSEGV`). `NULL` is conventionally address `0`, which the OS deliberately
leaves unmapped specifically so dereferencing a null pointer reliably
crashes instead of corrupting memory silently — that reliability is *why*
`if (ptr == NULL)` checks are effective as a safety net.

**Array decay** is a compile-time rule, not a runtime conversion: whenever
an array name appears in most expressions, the compiler substitutes the
address of element 0 in its place, because arrays and pointers use
compatible representations at the machine level — an array's "value" in an
expression context *is* a base address. This is also why pointer
arithmetic is type-aware: `numbers + 1` doesn't add 1 byte, it adds
`1 * sizeof(int)` bytes (4, typically), because the compiler scales the
offset by the pointee type's size so that `*(numbers + i)` lands exactly on
element `i`'s first byte — the same address arithmetic explained for
arrays in [Module 5](05-arrays-strings.md).

`swap(&x, &y)` demonstrates why passing addresses defeats pass-by-value:
`a` and `b` inside `swap` are still copies (of the *addresses*), but
dereferencing a copied address still reaches the original memory it points
to — `*a = *b` is a memory write at `x`'s actual stack location in `main`'s
frame, not a write to any copy. A pointer-to-pointer (`int **ptrToPtr`)
simply repeats this once more: it's a variable holding the address of
another variable (`ptr`) that itself holds an address — `**ptrToPtr`
dereferences twice, following two address hops in sequence to finally land
on `value`'s actual storage.

## Exercise

Write a program that declares an array of 5 integers. Using only pointer
notation (no `[]` indexing), write a loop that prints every element and its
memory address. Then write a function `void doubleValue(int *n)` that doubles
whatever integer its pointer points to, and call it on one of the array
elements (by passing `&array[i]`) to confirm the array itself changed.
