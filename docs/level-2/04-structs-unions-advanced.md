# 04 · Structs & Unions Advanced

[Level 1, Module 7](../level-1/07-structs.md) covered structs as record types:
group some fields, give them a name, pass them around. This module goes under
the hood — how the compiler lays a struct out in memory, why `sizeof` is often
bigger than you expect, and the three sibling constructs (`union`, bit-fields,
and the tagged-union idiom) that let a struct mean different things at
different times.

You need this the moment you start reading binary formats, writing network
code, or building linked data structures — all of which show up in the rest of
this level.

## Pointers to structs and the `->` operator

Once structs get big, you pass pointers instead of copies. `p->field` is
shorthand for `(*p).field`:

```c
#include <stdio.h>
#include <string.h>

typedef struct {
    char   name[32];
    int    quantity;
    double price;
} Item;

// const Item * = "I will read this and not modify it"
void print_item(const Item *it) {
    printf("%-10s qty=%-4d $%.2f\n", it->name, it->quantity, it->price);
}

// Non-const pointer: this one modifies the caller's struct.
void restock(Item *it, int amount) {
    it->quantity += amount;
}

int main(void) {
    Item widget = {"widget", 5, 9.99};

    print_item(&widget);
    restock(&widget, 10);
    print_item(&widget);

    // (*p).field and p->field are exactly equivalent
    Item *p = &widget;
    printf("%d == %d\n", (*p).quantity, p->quantity);
    return 0;
}
// Output:
// widget     qty=5    $9.99
// widget     qty=15   $9.99
// 15 == 15
```

Passing by pointer copies 8 bytes instead of 48, and it's the only way for the
function to modify the caller's struct — the same pass-by-value rule from
[Level 1, Module 4](../level-1/04-functions.md) applies to structs.

## Padding and alignment: why `sizeof` surprises you

The CPU wants each field at an address that's a multiple of its size. The
compiler inserts invisible **padding** bytes to make that happen:

```c
#include <stdio.h>
#include <stddef.h>

struct Bad  { char a; int b; char c; };   // char, pad, int, char, pad
struct Good { int b; char a; char c; };   // int, char, char, pad

int main(void) {
    printf("sizeof(struct Bad)  = %zu\n", sizeof(struct Bad));
    printf("sizeof(struct Good) = %zu\n", sizeof(struct Good));

    printf("Bad:  a@%zu b@%zu c@%zu\n",
           offsetof(struct Bad, a),
           offsetof(struct Bad, b),
           offsetof(struct Bad, c));
    printf("Good: b@%zu a@%zu c@%zu\n",
           offsetof(struct Good, b),
           offsetof(struct Good, a),
           offsetof(struct Good, c));
    return 0;
}
// Output on a typical 64-bit system:
// sizeof(struct Bad)  = 12
// sizeof(struct Good) = 8
// Bad:  a@0 b@4 c@8
// Good: b@0 a@4 c@5
```

Same three fields, same data, 50% more memory — purely because of declaration
order. `struct Bad` puts 3 padding bytes after `a` so `b` lands on a 4-byte
boundary, then pads the tail so an *array* of these structs keeps every element
aligned.

Practical rules:

- **Declare wide members first** (`double`, `long`, pointers), narrow ones
  last. This usually minimizes padding for free.
- `offsetof` (from `<stddef.h>`) tells you where a member actually sits.
- **Never assume a struct's byte layout is portable.** Padding differs between
  compilers and architectures, which is exactly why writing a raw struct to a
  file is risky — see [Module 5](05-binary-file-io.md).
- Padding bytes have **indeterminate values**, so comparing two structs with
  `memcmp` can report "different" for structs whose fields are all equal.
  Compare field by field.

## Self-referential structs

A struct cannot contain itself (infinite size), but it can contain a *pointer*
to itself. This is the foundation of every linked structure:

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct Node {          // tag "Node" is required to refer to itself
    int value;
    struct Node *next;         // note: "struct Node", the typedef isn't ready yet
} Node;

int main(void) {
    Node *head = NULL;

    // Build 3 -> 2 -> 1 by pushing onto the front
    for (int i = 1; i <= 3; i++) {
        Node *n = malloc(sizeof *n);
        if (n == NULL) return 1;
        n->value = i;
        n->next  = head;
        head = n;
    }

    for (Node *cur = head; cur != NULL; cur = cur->next) {
        printf("%d -> ", cur->value);
    }
    printf("NULL\n");

    // Free the list: save "next" BEFORE freeing the node
    Node *cur = head;
    while (cur != NULL) {
        Node *next = cur->next;
        free(cur);
        cur = next;
    }
    return 0;
}
// Output:
// 3 -> 2 -> 1 -> NULL
```

The struct **tag** (`struct Node`) is mandatory here: inside the braces the
`typedef` name doesn't exist yet. And notice the free loop saves `cur->next`
first — reading `cur->next` after `free(cur)` is a use-after-free, the bug from
[Module 2](02-dynamic-memory.md).

You'll build a full linked list in [the Level 2 project](10-project-inventory-system.md).

## Unions: one block of memory, several interpretations

A `union` gives every member the *same* starting address. Its size is that of
its largest member, and **only one member is meaningful at a time**:

```c
#include <stdio.h>

union Value {
    int    i;
    float  f;
    char   bytes[4];
};

int main(void) {
    union Value v;

    printf("sizeof(union Value) = %zu\n", sizeof v);

    v.i = 1234567;
    printf("as int:   %d\n", v.i);

    v.f = 3.14f;                  // OVERWRITES the same bytes
    printf("as float: %.2f\n", v.f);
    printf("as int:   %d   <-- meaningless now\n", v.i);

    return 0;
}
// Output:
// sizeof(union Value) = 4
// as int:   1234567
// as float: 3.14
// as int:   1078523331   <-- meaningless now
```

Writing one member and reading another is "type punning". C explicitly allows
it through a union (unlike C++), and it's how you'd inspect the raw bytes of a
float — but the result depends on endianness and float representation, so treat
it as machine-specific.

## Tagged unions: the safe way to use them

A bare union has no idea which member is live. The fix is to pair it with an
enum that records the current type:

```c
#include <stdio.h>

typedef enum { VAL_INT, VAL_DOUBLE, VAL_STRING } ValueType;

typedef struct {
    ValueType type;              // the "tag" -- says which member is valid
    union {
        int         i;
        double      d;
        const char *s;
    } as;                        // a union member named "as": v->as.i, v->as.d
} Variant;

void print_variant(const Variant *v) {
    switch (v->type) {
        case VAL_INT:    printf("int:    %d\n",    v->as.i); break;
        case VAL_DOUBLE: printf("double: %.2f\n",  v->as.d); break;
        case VAL_STRING: printf("string: %s\n",    v->as.s); break;
    }
}

int main(void) {
    Variant vals[] = {
        { VAL_INT,    { .i = 42 } },
        { VAL_DOUBLE, { .d = 2.5 } },
        { VAL_STRING, { .s = "hello" } }
    };

    for (int i = 0; i < 3; i++) print_variant(&vals[i]);

    printf("sizeof(Variant) = %zu\n", sizeof(Variant));
    return 0;
}
// Output:
// int:    42
// double: 2.50
// string: hello
// sizeof(Variant) = 16
```

This is how interpreters store dynamically-typed values, how `libpng` and
similar libraries model "one of several chunk types", and how you'd represent a
JSON value in C. Reading the wrong member is now a *bug you can catch* rather
than silent garbage.

Note the `{ .i = 42 }` syntax — **designated initializers**, added in C99. They
also work for structs and are far more readable than positional ones:

```c
Item it = { .price = 9.99, .quantity = 3, .name = "bolt" };   // order-free
```

## Bit-fields: packing flags into single bits

When memory really matters — embedded work, protocol headers — you can declare
members that occupy a specified number of *bits*:

```c
#include <stdio.h>

typedef struct {
    unsigned int is_active   : 1;   // 1 bit: 0 or 1
    unsigned int is_visible  : 1;
    unsigned int priority    : 3;   // 3 bits: 0..7
    unsigned int reserved    : 27;
} Flags;

int main(void) {
    Flags f = { .is_active = 1, .is_visible = 0, .priority = 5 };

    printf("active=%u visible=%u priority=%u\n",
           f.is_active, f.is_visible, f.priority);
    printf("sizeof(Flags) = %zu\n", sizeof f);

    f.priority = 9;    // 9 needs 4 bits -- silently truncated to 3 bits
    printf("priority after setting 9: %u\n", f.priority);
    return 0;
}
// Output:
// active=1 visible=0 priority=5
// sizeof(Flags) = 4
// priority after setting 9: 1
```

Bit-fields are convenient but **not portable in layout**: the standard leaves
bit ordering, straddling, and padding up to the implementation. Use them for
your own in-memory flags; use explicit shifts and masks (covered in
[Level 3, Module 4](../level-3/04-bit-manipulation.md)) when the exact bit
positions must match a file format or wire protocol.

## Quick reference

| Construct | Size | Members live at once | Typical use |
|-----------|------|----------------------|-------------|
| `struct` | sum of members + padding | all | records, nodes, objects |
| `union` | size of largest member | exactly one | type punning, memory reuse |
| tagged union | enum + union | one, tracked by the tag | variants, interpreters, parsers |
| bit-field | packed into words | all | flags, tight embedded state |
| `enum` | an `int` in practice | — | named constants, the tag itself |

## Exercise

Model a small shape system. Define `typedef enum { CIRCLE, RECTANGLE, TRIANGLE
} ShapeKind;` and a tagged union `Shape` holding the tag plus a union of
`{ double radius; }`, `{ double w, h; }`, and `{ double base, height; }`. Write
`double area(const Shape *s)` that switches on the tag, and
`void describe(const Shape *s)` that prints the kind and area. Build an array of
five shapes using designated initializers, print each, and print the total area.
Then add a `printf` of `sizeof(Shape)` and explain to yourself why it is what it
is, using the padding rules above.
