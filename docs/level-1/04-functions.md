# 04 · Functions

Functions let you break a program into named, reusable pieces. C's model is
simpler than object-oriented languages — there are no methods attached to
objects, just plain functions that take arguments and return a value.

## Declaration vs. definition

A **declaration** (also called a prototype) tells the compiler a function's
name, return type, and parameter types, without providing a body. A
**definition** provides the actual body — the code that runs.

```c
#include <stdio.h>

// Declaration (prototype) -- tells the compiler this function exists
int add(int a, int b);

int main(void) {
    int result = add(3, 4);   // compiler already knows add's signature
    printf("%d\n", result);   // 7
    return 0;
}

// Definition -- the actual implementation, can come after main
int add(int a, int b) {
    return a + b;
}
```

### Why prototypes matter

The compiler reads a file top to bottom. Without a prototype declared before
`main`, calling `add` there would fail because the compiler hasn't seen
`add`'s signature yet and can't check the call is correct. Prototypes solve
this by declaring the shape of a function up front, letting the actual
definition live anywhere — often in a different file entirely.

This is exactly why header files (`.h`) exist: they hold prototypes so
multiple `.c` files can share and call each other's functions without needing
to see each other's full source. We cover multi-file projects properly in
[Module 9](09-preprocessor-multifile.md).

## Parameters and return values

```c
#include <stdio.h>

double average(int a, int b, int c) {
    return (a + b + c) / 3.0;   // 3.0, not 3, to force floating-point division
}

void greet(const char *name) {   // void -- returns nothing
    printf("Hello, %s!\n", name);
}

int main(void) {
    printf("%.2f\n", average(4, 7, 9));   // 6.67
    greet("Sam");                          // Hello, Sam!
    return 0;
}
```

A function with return type `void` doesn't return a value — it's called
purely for its side effects (like printing).

## Pass-by-value semantics

When you call a function in C, each argument's *value* is copied into the
function's parameter. The function works on its own local copy — changes
inside the function never affect the caller's original variable.

```c
#include <stdio.h>

void increment(int n) {
    n = n + 1;          // only changes the local copy
    printf("Inside: %d\n", n);
}

int main(void) {
    int x = 5;
    increment(x);
    printf("Outside: %d\n", x);
    // Output:
    // Inside: 6
    // Outside: 5  -- x in main is untouched
    return 0;
}
```

This is a real limitation if you need a function to modify the caller's
variable directly. The fix is passing a *pointer* to the variable instead of
the variable itself — covered in [Module 6](06-pointers-basics.md), which
changes this story considerably (pointers let a function reach back and
modify the caller's memory on purpose).

## Recursion

A function calling itself, with a base case that stops the recursion:

```c
#include <stdio.h>

int factorial(int n) {
    if (n <= 1) {
        return 1;             // base case -- stops the recursion
    }
    return n * factorial(n - 1);   // recursive case
}

int main(void) {
    printf("%d\n", factorial(5));   // 120  (5*4*3*2*1)
    return 0;
}
```

```c
// Fibonacci, another classic recursion example
int fibonacci(int n) {
    if (n <= 1) {
        return n;
    }
    return fibonacci(n - 1) + fibonacci(n - 2);
}

int main(void) {
    for (int i = 0; i < 8; i++) {
        printf("%d ", fibonacci(i));
    }
    // Output: 0 1 1 2 3 5 8 13
    return 0;
}
```

Every recursive call without a reachable base case eventually crashes the
program with a stack overflow — always make sure the recursive case moves
toward the base case.

## Scope: local vs. global variables

```c
#include <stdio.h>

int counter = 0;   // global -- visible to every function in this file

void increment_counter(void) {
    counter++;      // modifies the global directly
}

int main(void) {
    int local = 100;   // local -- only visible inside main

    increment_counter();
    increment_counter();
    printf("%d\n", counter);   // 2

    // printf("%d\n", local);  -- fine here, but invisible to other functions
    return 0;
}
```

Global variables are visible everywhere in the file (and other files, if
declared `extern` — see Level 2), which makes them convenient but also risky:
any function can change them, making bugs harder to trace. Prefer local
variables and passing values explicitly unless you have a good reason.

## `static` for function-local persistent state

A local variable normally resets every time its function is called. Marking
it `static` makes it keep its value between calls instead:

```c
#include <stdio.h>

void call_counter(void) {
    static int calls = 0;   // initialized once, persists across calls
    calls++;
    printf("Called %d time(s)\n", calls);
}

int main(void) {
    call_counter();   // Called 1 time(s)
    call_counter();   // Called 2 time(s)
    call_counter();   // Called 3 time(s)
    return 0;
}
```

Unlike a global, a `static` local variable is still only visible inside the
function it's declared in — you get persistence without exposing it to the
rest of the file.

| Concept | Meaning |
|---------|---------|
| Declaration / prototype | Tells the compiler a function's signature exists, no body |
| Definition | The actual function body |
| Pass-by-value | Arguments are copied; the callee can't modify the caller's original |
| Local variable | Scoped to its function/block, reset each call |
| Global variable | Visible to the whole file, persists for the program's lifetime |
| `static` local | Scoped to its function, but persists between calls |

## Exercise

Write a function `int power(int base, int exponent)` that computes
`base^exponent` using recursion (base case: `exponent == 0` returns `1`).
Then write a function `void track_calls(void)` using a `static` local counter
that prints how many times it has been called so far. Call `power` a few
times with different arguments, then call `track_calls` three times in a row
and confirm the count increases each time.
