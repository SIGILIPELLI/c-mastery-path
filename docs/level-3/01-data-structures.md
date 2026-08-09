# 01 · Data Structures in C

Every data structure you've used in a higher-level language — Python's
`list`, Java's `ArrayDeque`, JavaScript's array — is built out of the same
handful of primitives you already have: structs, pointers, and `malloc`.
This module builds three of the most common ones from scratch: the
**singly linked list**, the **stack**, and the **queue**. Understanding how
they're built (and what can go wrong) is what makes Level 3's trees, graphs,
and the final key-value store project make sense later.

## The linked list: nodes chained by pointers

An array stores elements contiguously — fast random access, but resizing
means copying everything. A linked list stores each element in its own
heap-allocated **node**, plus a pointer to the next node. Growing it never
requires moving existing elements.

```c
// list1.c
#include <stdio.h>
#include <stdlib.h>

typedef struct Node {
    int value;
    struct Node *next;
} Node;

Node *push_front(Node *head, int value) {
    Node *n = malloc(sizeof(Node));
    if (!n) {
        fprintf(stderr, "out of memory\n");
        exit(1);
    }
    n->value = value;
    n->next = head;      // new node points at the old head
    return n;             // caller must update its head pointer to this
}

void print_list(const Node *head) {
    for (const Node *cur = head; cur != NULL; cur = cur->next) {
        printf("%d -> ", cur->value);
    }
    printf("NULL\n");
}

void free_list(Node *head) {
    while (head != NULL) {
        Node *next = head->next;  // save before freeing head
        free(head);
        head = next;
    }
}

int main(void) {
    Node *head = NULL;
    head = push_front(head, 30);
    head = push_front(head, 20);
    head = push_front(head, 10);

    print_list(head);
    free_list(head);
    head = NULL;
    return 0;
}
```

```bash
gcc -Wall -Wextra -g -O0 -fsanitize=address,undefined -o list1 list1.c
./list1
```

```text
10 -> 20 -> 30 -> NULL
```

A `struct` referring to itself (`struct Node *next` inside `struct Node`)
works because the member is a *pointer*, not the struct by value — the
compiler knows a pointer's size regardless of what it points to. A struct
directly containing a field of its own type wouldn't compile: the compiler
couldn't compute a finite size for it.

### The self-inflicted bugs of linked lists

Three mistakes account for almost every linked-list bug:

1. **Forgetting to reassign `head`.** `push_front` returns the new head —
   if you call it and discard the return value, the node you just allocated
   is unreachable (and leaked) while `head` still points at the old list.
2. **Freeing a node before reading `next`.** `free(cur); cur = cur->next;`
   reads freed memory. `free_list` above saves `next` *before* calling
   `free`, specifically to avoid this.
3. **Using a pointer after `free`ing it** ("use-after-free"). Once you
   `free(node)`, `node` is a dangling pointer — the memory may already be
   reused by something else. Set it to `NULL` immediately after freeing if
   you're not about to reassign it, so any accidental reuse crashes loudly
   instead of corrupting data silently.

## The stack: last in, first out

A stack only ever grows or shrinks from one end (the "top"). An array with a
`top` index is a simple, cache-friendly implementation when you know a
reasonable maximum size up front:

```c
// stack.c
#include <stdio.h>

#define STACK_CAP 8

typedef struct {
    int data[STACK_CAP];
    int top;   // index of the next free slot; 0 means empty
} Stack;

void stack_init(Stack *s) {
    s->top = 0;
}

int stack_push(Stack *s, int value) {
    if (s->top >= STACK_CAP) {
        return 0;   // full
    }
    s->data[s->top] = value;
    s->top++;
    return 1;
}

int stack_pop(Stack *s, int *out) {
    if (s->top == 0) {
        return 0;   // empty
    }
    s->top--;
    *out = s->data[s->top];
    return 1;
}

int main(void) {
    Stack s;
    stack_init(&s);

    stack_push(&s, 1);
    stack_push(&s, 2);
    stack_push(&s, 3);

    int value;
    while (stack_pop(&s, &value)) {
        printf("popped %d\n", value);
    }

    if (!stack_pop(&s, &value)) {
        printf("stack is empty\n");
    }
    return 0;
}
```

```text
popped 3
popped 2
popped 1
stack is empty
```

Notice both `stack_push` and `stack_pop` return an `int` status rather than
silently doing nothing on failure. In C there's no exception to throw when
you push past capacity — callers who ignore the return value get to fail
just as silently, which is why checking it matters more here than in
languages with exceptions.

Function call stacks, undo history, and depth-first traversal (Module 02)
all use exactly this last-in-first-out shape.

## The queue: first in, first out

A queue serves elements in the order they arrived. Popping from the front of
a plain array would mean shifting every remaining element down by one —
O(n) per dequeue. A **circular buffer** avoids that by wrapping the index
around with `%` instead of shifting memory:

```c
// queue.c
#include <stdio.h>

#define QUEUE_CAP 4

typedef struct {
    int data[QUEUE_CAP];
    int head;   // index of the oldest element
    int count;  // number of elements currently stored
} Queue;

void queue_init(Queue *q) {
    q->head = 0;
    q->count = 0;
}

int queue_enqueue(Queue *q, int value) {
    if (q->count == QUEUE_CAP) {
        return 0;   // full
    }
    int tail = (q->head + q->count) % QUEUE_CAP;
    q->data[tail] = value;
    q->count++;
    return 1;
}

int queue_dequeue(Queue *q, int *out) {
    if (q->count == 0) {
        return 0;   // empty
    }
    *out = q->data[q->head];
    q->head = (q->head + 1) % QUEUE_CAP;
    q->count--;
    return 1;
}

int main(void) {
    Queue q;
    queue_init(&q);

    queue_enqueue(&q, 100);
    queue_enqueue(&q, 200);
    queue_enqueue(&q, 300);

    int value;
    queue_dequeue(&q, &value);
    printf("dequeued %d\n", value);

    // Enqueue two more -- this wraps the circular buffer past the end of data[]
    queue_enqueue(&q, 400);
    queue_enqueue(&q, 500);

    while (queue_dequeue(&q, &value)) {
        printf("dequeued %d\n", value);
    }
    return 0;
}
```

```text
dequeued 100
dequeued 200
dequeued 300
dequeued 400
dequeued 500
```

Walk the trace: capacity is 4, so after dequeuing `100` the buffer holds
`{_, 200, 300, _}` with `head = 1, count = 2`. Enqueuing `400` writes it at
index `3` (`(1 + 2) % 4`), and enqueuing `500` wraps around to index `0`
(`(1 + 3) % 4`) — physically *before* `200` and `300` in memory, but
logically after them. That wraparound is the entire trick of a circular
buffer, and it's the same technique behind ring buffers used in audio and
networking code.

## Singly linked list vs. array-backed stack/queue

| | Linked list | Array (stack/queue) |
|---|---|---|
| Insert at the tracked end | O(1), no resize ever needed | O(1) until full, then requires a resize (or a fixed cap that rejects) |
| Random access by index | O(n) — must walk from head | O(1) — direct indexing |
| Memory layout | Scattered on the heap, one allocation per node | Contiguous — much better cache locality |
| Per-element overhead | One pointer per node (8 bytes on a 64-bit machine) | None beyond the data itself |
| Typical use | Unknown/unbounded size, frequent insert/delete at ends | Known upper bound, performance-sensitive access |

## Exercise

Extend `stack.c` into a **balanced-parentheses checker**: write a function
`int is_balanced(const char *expr)` that pushes an opening bracket
(`(`, `[`, `{`) onto the stack as it scans `expr`, and on a closing bracket,
pops and checks it matches. Return `1` if every bracket closes correctly and
the stack is empty at the end, `0` otherwise. Test it against `"(a[b]{c})"`
(balanced), `"(a[b)]"` (mismatched), and `"(a"` (unclosed). Then compile and
run it under `-fsanitize=address,undefined` to confirm you never pop from an
empty stack.
