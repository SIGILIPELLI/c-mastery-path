# 03 · Control Flow

Control flow statements decide which code runs, and how many times. C's set
will look familiar if you've seen any C-family language, since most of them
borrowed this exact syntax.

## if / else if / else

```c
#include <stdio.h>

int main(void) {
    int score = 72;

    if (score >= 90) {
        printf("Grade: A\n");
    } else if (score >= 80) {
        printf("Grade: B\n");
    } else if (score >= 70) {
        printf("Grade: C\n");
    } else {
        printf("Grade: F\n");
    }
    // Output: Grade: C

    return 0;
}
```

Conditions are evaluated top to bottom; the first branch whose condition is
true runs, and the rest are skipped. In C, any non-zero value is treated as
"true" and `0` is "false" — there's no separate boolean type built into the
language pre-C99 (C99 added `_Bool` via `<stdbool.h>`, which most modern code
uses).

## Ternary operator

A compact form of if/else that produces a value, useful for simple
either/or expressions:

```c
int a = 10, b = 3;
int max = (a > b) ? a : b;
printf("%d\n", max);   // 10

// Equivalent to:
// int max;
// if (a > b) { max = a; } else { max = b; }
```

## switch statement

`switch` compares one value against several constant cases — useful when
you'd otherwise write a long `else if` chain against the same variable.

```c
#include <stdio.h>

int main(void) {
    int day = 3;

    switch (day) {
        case 1:
            printf("Monday\n");
            break;
        case 2:
            printf("Tuesday\n");
            break;
        case 3:
            printf("Wednesday\n");
            break;
        default:
            printf("Some other day\n");
    }
    // Output: Wednesday

    return 0;
}
```

### Fallthrough

If you omit `break`, execution "falls through" into the next case instead of
exiting the switch. This trips up nearly every C beginner at least once:

```c
#include <stdio.h>

int main(void) {
    int n = 1;

    switch (n) {
        case 1:
            printf("one\n");
            // no break -- falls through!
        case 2:
            printf("two\n");
            break;
        case 3:
            printf("three\n");
            break;
    }
    // Output:
    // one
    // two
    return 0;
}
```

Fallthrough is almost always a footgun when it's accidental — forgetting a
`break` silently changes behavior with no compiler error. But it's
occasionally used *intentionally*, when several cases should share the same
handling:

```c
switch (grade) {
    case 'A':
    case 'B':
    case 'C':
        printf("Passing\n");   // A, B, and C all fall into this one line
        break;
    default:
        printf("Not passing\n");
}
```

When you do this on purpose, a `// fallthrough` comment is good practice so
the next reader knows it wasn't a mistake.

## while loop

Runs as long as the condition stays true, checked before each iteration:

```c
#include <stdio.h>

int main(void) {
    int i = 0;
    while (i < 5) {
        printf("%d\n", i);
        i++;
    }
    // Output: 0 1 2 3 4
    return 0;
}
```

## do-while loop

Same as `while`, but the condition is checked *after* the body runs — so the
body always executes at least once, even if the condition is false from the
start:

```c
#include <stdio.h>

int main(void) {
    int i = 10;
    do {
        printf("%d\n", i);
        i++;
    } while (i < 5);
    // Output: 10 -- runs once even though 10 < 5 is false
    return 0;
}
```

## for loop

Bundles initialization, condition, and increment into one line — the natural
choice when you know in advance how many times to loop:

```c
#include <stdio.h>

int main(void) {
    for (int i = 0; i < 5; i++) {
        printf("%d\n", i);
    }
    // Output: 0 1 2 3 4
    return 0;
}
```

## break and continue

`break` exits a loop (or switch) immediately. `continue` skips the rest of the
current iteration and jumps to the next one.

```c
#include <stdio.h>

int main(void) {
    for (int i = 0; i < 10; i++) {
        if (i == 5) {
            break;          // stop the loop entirely once i hits 5
        }
        if (i % 2 == 0) {
            continue;       // skip printing even numbers
        }
        printf("%d\n", i);
    }
    // Output: 1 3
    return 0;
}
```

## Exercise

Write a program that loops from 1 to 30 with a `for` loop. For each number:
print "Fizz" if divisible by 3, "Buzz" if divisible by 5, "FizzBuzz" if
divisible by both, and the number itself otherwise. Use `continue` to skip
printing the number 13 entirely (superstition demands it), and use a `switch`
with intentional fallthrough somewhere to print "small" for numbers 1, 2, and
3 the first time you encounter them.
