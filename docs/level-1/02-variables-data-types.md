# 02 · Variables, Data Types & Operators

## Declaring and initializing variables

Unlike dynamically typed languages, C requires you to state a variable's type
up front — the compiler uses that type to decide how many bytes to allocate
and how to interpret them.

```c
#include <stdio.h>

int main(void) {
    int age = 30;              // whole numbers
    float price = 19.99f;      // single-precision floating point
    double pi = 3.14159265;    // double-precision floating point
    char grade = 'B';          // a single character, stored as a small integer
    int score;                 // declared, not yet initialized -- holds garbage until assigned
    score = 100;

    printf("%d %.2f %f %c %d\n", age, price, pi, grade, score);
    return 0;
}
// Output:
// 30 19.99 3.141593 B 100
```

Reading an uninitialized variable's value before assigning it is undefined
behavior in C — the compiler doesn't zero it out for you. Always initialize
before use.

## Primitive types

C's built-in types map fairly directly onto the machine's memory:

```c
int whole = 42;
short small = 100;
long big = 1000000L;
long long huge = 10000000000LL;
unsigned int positive_only = 4000000000u;
float f = 1.5f;
double d = 1.5;
char c = 'x';
```

`unsigned` variants store only non-negative numbers, which doubles the top end
of the positive range in exchange for giving up negative values entirely.

### `sizeof` — how big is this type, really?

Exact sizes aren't guaranteed by the C standard — they depend on the platform
and compiler — so `sizeof` lets you ask at compile time instead of assuming:

```c
#include <stdio.h>

int main(void) {
    printf("int: %zu bytes\n", sizeof(int));
    printf("float: %zu bytes\n", sizeof(float));
    printf("double: %zu bytes\n", sizeof(double));
    printf("char: %zu bytes\n", sizeof(char));
    printf("long: %zu bytes\n", sizeof(long));
    return 0;
}
// Output (typical on a 64-bit machine):
// int: 4 bytes
// float: 4 bytes
// double: 8 bytes
// char: 1 bytes
// long: 8 bytes
```

`%zu` is the format specifier for `size_t`, the unsigned integer type
`sizeof` returns.

### Cheat sheet: common types

| Type | Typical size | Typical range |
|------|--------------|---------------|
| `char` | 1 byte | -128 to 127 (or 0 to 255 if unsigned) |
| `short` | 2 bytes | -32,768 to 32,767 |
| `int` | 4 bytes | -2,147,483,648 to 2,147,483,647 |
| `unsigned int` | 4 bytes | 0 to 4,294,967,295 |
| `long` | 8 bytes (4 on some platforms) | roughly ±9.2 × 10^18 |
| `long long` | 8 bytes | roughly ±9.2 × 10^18 |
| `float` | 4 bytes | ~7 significant decimal digits |
| `double` | 8 bytes | ~15 significant decimal digits |

Treat these sizes as "typical, not guaranteed" — if exact width matters (for
file formats or network protocols), Level 2 covers the fixed-width types like
`int32_t` from `<stdint.h>`.

## Type casting

C sometimes converts types for you (implicit conversion), and sometimes you
have to ask for it explicitly.

```c
#include <stdio.h>

int main(void) {
    int a = 7;
    int b = 2;

    // Implicit: both operands are int, so this is integer division
    printf("%d\n", a / b);          // 3 -- fraction is discarded, not rounded

    // Explicit cast: force one operand to double before dividing
    printf("%f\n", (double)a / b);  // 3.500000

    double price = 9.75;
    int whole_dollars = (int)price;  // explicit cast, truncates toward zero
    printf("%d\n", whole_dollars);   // 9

    return 0;
}
```

Implicit conversions happen automatically when types mix in an expression
(e.g. `int` combined with `double` promotes the `int` to `double` first).
Explicit casts, written as `(type)value`, are how you override the default and
tell the compiler exactly what conversion you want — useful for avoiding
surprises like integer division when you meant real division.

## Operators overview

```c
#include <stdio.h>

int main(void) {
    int a = 10, b = 3;

    // Arithmetic
    printf("%d %d %d %d %d\n", a + b, a - b, a * b, a / b, a % b);
    // 13 7 30 3 1  -- % is remainder, not "percent"

    // Relational -- produce 0 (false) or 1 (true)
    printf("%d %d %d\n", a > b, a == b, a != b);
    // 1 0 1

    // Logical
    int x = 1, y = 0;
    printf("%d %d %d\n", x && y, x || y, !x);
    // 0 1 0

    return 0;
}
```

### Bitwise operators (brief preview)

C also has operators that act directly on the binary representation of
integers: `&` (AND), `|` (OR), `^` (XOR), `~` (NOT), `<<` (left shift), and
`>>` (right shift). These are used heavily for flags, masks, and
low-level tricks:

```c
int flags = 0b0101;   // binary literal: 5
int mask  = 0b0011;   // 3
printf("%d\n", flags & mask);   // 1 -- bits set in both
printf("%d\n", flags | mask);   // 7 -- bits set in either
printf("%d\n", flags << 1);     // 10 -- shift left, multiply by 2
```

That's just enough to recognize them when you see them — a full treatment,
including practical bit-manipulation patterns, is in
[Level 4](../level-4/06-advanced-data-structures.md).

## Integer overflow

Integers have a fixed number of bits, so arithmetic that exceeds the type's
range wraps around silently instead of raising an error:

```c
#include <stdio.h>
#include <limits.h>

int main(void) {
    int max = INT_MAX;          // largest value an int can hold
    printf("%d\n", max);        // 2147483647
    printf("%d\n", max + 1);    // -2147483648 -- wraps around to the minimum!
    return 0;
}
```

This is a common source of subtle bugs — C will not warn you at runtime.
Choosing a wider type (`long`, `long long`) or an unsigned type buys more
headroom but doesn't eliminate the problem, it just moves the boundary.

## Exercise

Write a program that declares an `int`, a `float`, a `double`, and a `char`,
prints the `sizeof` each one, then performs an explicit cast dividing two
integer variables to get a precise decimal result (not truncated integer
division). Finally, declare an `int` set to `INT_MAX` (from `<limits.h>`) and
print what happens when you add `1` to it.
