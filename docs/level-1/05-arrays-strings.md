# 05 · Arrays & Strings

Arrays hold a fixed-size, contiguous block of same-typed values. C strings are
just arrays of `char` with a special end-of-string marker — there is no
built-in string type.

## Declaring and using arrays

```c
#include <stdio.h>

int main(void) {
    int scores[5] = {90, 85, 77, 92, 68};

    printf("%d\n", scores[0]);   // 90 -- indexing starts at 0
    printf("%d\n", scores[4]);   // 68 -- last valid index is size - 1

    scores[2] = 80;              // arrays are mutable
    printf("%d\n", scores[2]);   // 80

    int sum = 0;
    for (int i = 0; i < 5; i++) {
        sum += scores[i];
    }
    printf("sum = %d\n", sum);   // sum = 415

    return 0;
}
```

C does **not** check array bounds for you. Reading or writing `scores[5]` or
`scores[-1]` compiles and often "works" until it silently corrupts nearby
memory or crashes — always track the size yourself (e.g. with a `#define` or
a separate `size_t count` variable) and stay inside it.

## Strings are `char` arrays

A C string is a `char` array ending in a null terminator, `'\0'`:

```c
#include <stdio.h>
#include <string.h>

int main(void) {
    char name[20] = "Ada";   // compiler adds the '\0' automatically

    printf("%s\n", name);          // Ada
    printf("%zu\n", strlen(name)); // 3 -- length NOT counting '\0'
    printf("%zu\n", sizeof(name)); // 20 -- full buffer size

    return 0;
}
```

`sizeof` and `strlen` answer different questions: `sizeof` is the buffer's
total capacity; `strlen` is how many characters are actually in use before the
terminator.

## Common `<string.h>` functions

```c
#include <stdio.h>
#include <string.h>

int main(void) {
    char greeting[50] = "Hello";

    strcat(greeting, ", world!");         // append -- buffer must be big enough
    printf("%s\n", greeting);              // Hello, world!

    char copy[50];
    strcpy(copy, greeting);                // copy into another buffer
    printf("%s\n", copy);                  // Hello, world!

    if (strcmp("abc", "abc") == 0) {       // 0 means equal
        printf("equal\n");
    }

    printf("%zu\n", strlen(greeting));     // 13

    return 0;
}
```

`strcat` and `strcpy` do **not** check that the destination buffer is large
enough — writing past the end silently corrupts memory (a classic source of
real-world security bugs). Prefer the bounded versions `strncat`/`strncpy`
when the input size isn't fully controlled, and always size destination
buffers generously.

## Reading strings from input safely

```c
#include <stdio.h>

int main(void) {
    char name[50];

    printf("Enter your name: ");
    fgets(name, sizeof(name), stdin);   // safe: won't overflow the buffer

    // fgets keeps the trailing newline -- strip it if present
    size_t len = strlen(name);
    if (len > 0 && name[len - 1] == '\n') {
        name[len - 1] = '\0';
    }

    printf("Hello, %s!\n", name);
    return 0;
}
```

Avoid `gets()` entirely — it has no way to limit how much it reads and is
banned from modern C for that reason. `fgets` with an explicit buffer size is
the safe replacement.

## Multi-dimensional arrays

```c
#include <stdio.h>

int main(void) {
    int grid[2][3] = {
        {1, 2, 3},
        {4, 5, 6}
    };

    for (int row = 0; row < 2; row++) {
        for (int col = 0; col < 3; col++) {
            printf("%d ", grid[row][col]);
        }
        printf("\n");
    }
    // Output:
    // 1 2 3
    // 4 5 6

    return 0;
}
```

## Cheat sheet

| Task | Function/Syntax |
|------|------------------|
| Declare array | `int arr[5];` |
| Declare + initialize | `int arr[5] = {1,2,3,4,5};` |
| Array length (elements) | track it yourself, or `sizeof(arr)/sizeof(arr[0])` for stack arrays |
| String length (chars, no `\0`) | `strlen(s)` |
| Copy string | `strcpy(dst, src)` / `strncpy` (bounded) |
| Append string | `strcat(dst, src)` / `strncat` (bounded) |
| Compare strings | `strcmp(a, b) == 0` means equal |
| Read a line safely | `fgets(buf, sizeof(buf), stdin)` |

## Exercise

Write a program that declares a `char` buffer, reads a line of input into it
with `fgets`, strips the trailing newline, and then prints: the string itself,
its length via `strlen`, and the string reversed (build the reversed version
character-by-character into a second buffer — don't use any library reversal
function).
