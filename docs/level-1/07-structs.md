# 07 · Structs

C's built-in types (`int`, `double`, `char`, arrays) only get you so far when
modeling real-world data. A `struct` lets you bundle several related values
of different types together under one name, so a "point" or a "student" can
be passed around as a single unit instead of a scattered handful of loose
variables.

## Defining and declaring a struct

```c
#include <stdio.h>

struct Point {
    int x;
    int y;
};

int main(void) {
    struct Point p1;   // declare a variable of type "struct Point"
    p1.x = 3;          // access members with the dot operator
    p1.y = 7;

    printf("p1 = (%d, %d)\n", p1.x, p1.y);

    // Structs can also be initialized at declaration time
    struct Point p2 = {10, 20};
    printf("p2 = (%d, %d)\n", p2.x, p2.y);

    return 0;
}
// Output:
// p1 = (3, 7)
// p2 = (10, 20)
```

The `struct Point` declaration itself doesn't allocate any memory or create a
variable — it just describes the *shape* of the data. `struct Point p1;` is
what actually creates a variable with that shape, the same way `int x;`
creates an `int` variable after `int` is defined as a type by the language.

## The dot operator

Once you have a struct variable, `.` accesses (reads or writes) its members,
exactly like `p1.x` and `p1.y` above. Structs can be compared member-by-member
(there's no built-in `==` for whole structs), copied, and passed around:

```c
struct Point p3 = p2;   // copies all members: p3.x = 10, p3.y = 20
p3.x = 99;              // only p3 changes -- p2 is untouched
```

## Nesting: structs containing arrays and other structs

Struct members can themselves be arrays or other structs, which is how you
model richer records:

```c
#include <stdio.h>
#include <string.h>

struct Point {
    int x;
    int y;
};

struct Student {
    char name[50];        // fixed-size array member
    int age;
    double gpa;
    struct Point office;  // a nested struct
};

int main(void) {
    struct Student s;

    strcpy(s.name, "Maria Chen");   // can't assign strings directly to char arrays
    s.age = 21;
    s.gpa = 3.8;
    s.office.x = 4;      // reach into the nested struct with another dot
    s.office.y = 12;

    printf("%s, age %d, GPA %.1f, office at (%d, %d)\n",
           s.name, s.age, s.gpa, s.office.x, s.office.y);

    return 0;
}
// Output:
// Maria Chen, age 21, GPA 3.8, office at (4, 12)
```

Note that `s.name = "Maria Chen";` would **not** compile — arrays (including
`char` arrays used as strings) can't be assigned with `=` after declaration,
so `strcpy` is used instead, same as in
[Module 5, Arrays & Strings](05-arrays-strings.md).

## `typedef` — dropping the `struct` keyword

Writing `struct Point` everywhere is repetitive. `typedef` creates an alias so
you can just write `Point`:

```c
#include <stdio.h>

typedef struct {
    int x;
    int y;
} Point;   // "Point" is now a type name on its own

int main(void) {
    Point p1 = {3, 7};        // no "struct" keyword needed
    Point p2 = {10, 20};

    printf("p1 = (%d, %d)\n", p1.x, p1.y);
    printf("p2 = (%d, %d)\n", p2.x, p2.y);

    return 0;
}
// Output:
// p1 = (3, 7)
// p2 = (10, 20)
```

This pattern — an anonymous `struct` immediately given a name via `typedef` —
is by far the most common way structs are declared in real C code, and it's
what the rest of this module uses.

## Arrays of structs

Just like arrays of `int` or `char`, you can have arrays of structs — useful
any time you're managing a list of records (contacts, students, inventory
items):

```c
#include <stdio.h>
#include <string.h>

typedef struct {
    char name[50];
    double gpa;
} Student;

int main(void) {
    Student roster[3];

    strcpy(roster[0].name, "Alice");
    roster[0].gpa = 3.9;

    strcpy(roster[1].name, "Bob");
    roster[1].gpa = 3.2;

    strcpy(roster[2].name, "Carol");
    roster[2].gpa = 3.6;

    for (int i = 0; i < 3; i++) {
        printf("%-10s GPA: %.1f\n", roster[i].name, roster[i].gpa);
    }

    return 0;
}
// Output:
// Alice      GPA: 3.9
// Bob        GPA: 3.2
// Carol      GPA: 3.6
```

This exact pattern — an array of a struct with `char` fields — is the
backbone of the contact book you'll build in
[Module 10](10-project-contact-book.md).

## Passing structs to functions

Passing a struct to a function by value copies the *entire* struct, member by
member, into the function's parameter — just like passing an `int` copies the
value:

```c
#include <stdio.h>

typedef struct {
    int x;
    int y;
} Point;

void printPoint(Point p) {   // p is a full copy of whatever was passed in
    printf("(%d, %d)\n", p.x, p.y);
}

void tryToMove(Point p) {
    p.x += 100;   // modifies the local copy only
}

int main(void) {
    Point origin = {0, 0};

    printPoint(origin);   // (0, 0)
    tryToMove(origin);
    printPoint(origin);   // still (0, 0) -- tryToMove only changed its own copy

    return 0;
}
// Output:
// (0, 0)
// (0, 0)
```

For a small struct like `Point`, copying is cheap and harmless. For larger
structs, or when a function genuinely needs to modify the caller's struct,
you pass a **pointer to the struct** instead — combined with the `->` operator
to access members through the pointer without an explicit dereference. That's
a deliberate deep dive in
[Level 2, Module 4](../level-2/04-structs-unions-advanced.md), building on the
pointer basics from [Module 6](06-pointers-basics.md).

| Concept | Example | Meaning |
|---------|---------|---------|
| Define a struct type | `struct Point { int x; int y; };` | Describes a shape of grouped data |
| Declare a variable | `struct Point p;` | Creates a variable with that shape |
| Access a member | `p.x` | Dot operator reads/writes a field |
| `typedef` | `typedef struct {...} Point;` | Lets you write `Point` instead of `struct Point` |
| Nesting | `struct Student { struct Point office; };` | A struct field can itself be a struct |
| Pass by value | `void f(Point p)` | Function gets a full copy |

## Exercise

Define a `typedef struct` called `Rectangle` with members `width` and `height`
(both `double`), and a nested `Point topLeft` (reusing the `Point` struct from
this module) marking where the rectangle sits on a grid. Write a function
`double area(Rectangle r)` that returns `r.width * r.height`, and a function
`void printRectangle(Rectangle r)` that prints the rectangle's top-left
position, width, height, and area. Create an array of 3 `Rectangle` values in
`main`, fill them in, and print all three using your function.
