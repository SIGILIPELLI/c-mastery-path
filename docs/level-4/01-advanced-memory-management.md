# 01 · Advanced Memory Management

`malloc` is a general-purpose allocator, and general-purpose means it makes
no assumptions: any size, any lifetime, any order of frees, from any thread.
Paying for that generality on every one of two million small objects is
often the single largest cost in a C program.

A **custom allocator** trades generality for speed. If you know your objects
are all the same size, or that they all die at the same moment, you can
replace a few hundred instructions of bookkeeping with a pointer addition.
This module builds the two allocators that cover most real cases — the arena
and the pool — and then shows the price you pay for using them.

## Arena (bump) allocator: allocation is a pointer add

An arena grabs one big block up front and hands out slices by moving a
cursor forward. There is no per-object `free` at all; you release the whole
region at once.

```c
// arena.c -- a bump allocator: allocation is a pointer add, free is one reset
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>

typedef struct {
    unsigned char *base;
    size_t cap, used;
    size_t high_water;
} Arena;

static Arena arena_create(size_t cap) {
    Arena a = { malloc(cap), cap, 0, 0 };
    return a;
}

static void *arena_alloc(Arena *a, size_t size, size_t align) {
    /* round `used` up to the next multiple of align (align must be a power of 2) */
    size_t aligned = (a->used + (align - 1)) & ~(align - 1);
    if (aligned + size > a->cap) return NULL;      /* out of arena */
    void *p = a->base + aligned;
    a->used = aligned + size;
    if (a->used > a->high_water) a->high_water = a->used;
    return p;
}

static void arena_reset(Arena *a) { a->used = 0; }
```

The alignment arithmetic is the part that must be right.
`(used + align - 1) & ~(align - 1)` rounds up to the next multiple of a
power of two — [Level 3's bit manipulation
module](../level-3/04-bit-manipulation.md) covers why. Get it wrong and you
hand back a `double *` pointing at an odd address, which is **undefined
behaviour** even on x86 where the load happens to work; on some ARM
configurations and for SIMD types it faults outright.

```text
base       = 0x619000000580
name   +  0  align 1
vals   + 16  align 8  (padded past 16)
nums   + 40  align 4
used = 56 of 1024 bytes
data: "arena" 1.5 2.5 3.5 | 0 1 4 9
vals aligned to 8? yes
after reset: used = 0, high water = 56
oversized request -> NULL
```

`name` took bytes 0–15, so the cursor sat at 16 — already a multiple of 8,
so `vals` needed no padding and got offset 16. Three doubles ended at 40,
where `nums` landed. Total: 56 bytes, three allocations, and exactly **one**
`malloc` and one `free` in the entire program.

`_Alignof(T)` (C11) is how you ask for the right value rather than guessing
8. Never hardcode: `long double` and SIMD vector types want 16 on many
platforms.

Arenas fit anything with a clear lifetime boundary — per-request state in a
server, per-frame data in a renderer, an AST that dies when compilation
finishes. The `high_water` field is worth keeping: it tells you the smallest
arena you could have gotten away with.

## Pool allocator: fixed-size blocks and an intrusive free list

Arenas cannot free individual objects. When objects *do* die independently
but are all the same size, a **pool** gives you both — and the free list
costs zero extra memory, because it lives inside the free blocks themselves.

```c
// pool.c -- fixed-size block allocator with an intrusive free list
typedef struct {
    void *memory;
    void *free_list;      /* head of a list threaded through the free blocks */
    size_t block_size, nblocks, in_use;
} Pool;

static Pool pool_create(size_t block_size, size_t nblocks) {
    Pool p = { malloc(block_size * nblocks), NULL, block_size, nblocks, 0 };
    /* Link every block into the free list, back to front. */
    unsigned char *base = p.memory;
    for (size_t i = nblocks; i-- > 0; ) {
        void *blk = base + i * block_size;
        *(void **)blk = p.free_list;     /* store next-pointer INSIDE the block */
        p.free_list = blk;
    }
    return p;
}

static void *pool_alloc(Pool *p) {
    if (!p->free_list) return NULL;
    void *blk = p->free_list;
    p->free_list = *(void **)blk;        /* pop */
    p->in_use++;
    return blk;
}

static void pool_free(Pool *p, void *blk) {
    *(void **)blk = p->free_list;        /* push */
    p->free_list = blk;
    p->in_use--;
}
```

```text
sizeof(Node) = 64
allocated 3, in_use = 3
a=0 b=64 c=128 (offsets, block size 64)
freed b, in_use = 2
next alloc reused b's slot? yes (offset 64)
pool exhausted -> NULL
```

Allocation and deallocation are each a handful of instructions — pop or push
a linked list — with no size classes, no coalescing, no search. Freeing `b`
put its 64-byte slot at the head of the list, so the very next `pool_alloc`
handed the same address back.

Two constraints are baked into that trick:

- A block must be **at least `sizeof(void *)` bytes**, since a free block
  stores a pointer in its first 8 bytes. A pool of 4-byte blocks silently
  corrupts memory.
- Reading `*(void **)blk` on a block whose alignment is weaker than
  `_Alignof(void *)` is undefined behaviour. Always round `block_size` up to
  a multiple of `_Alignof(max_align_t)`.

## What this actually buys you

Same workload — two million small records, filled, summed, released — once
with `malloc`/`free` per object and once from an arena:

```c
// bench.c -- malloc/free per node vs one arena, same workload
for (int i = 0; i < N; i++) { ptrs[i] = malloc(sizeof(Rec)); ptrs[i]->a = i; }
...
for (int i = 0; i < N; i++) free(ptrs[i]);
```

versus

```c
for (int i = 0; i < N; i++) {
    ap[i] = (Rec *)(arena + used);
    used += sizeof(Rec);
    ap[i]->a = i;
}
...
free(arena);                        /* ONE free for 2,000,000 objects */
```

```bash
clang -Wall -Wextra -std=c11 -O2 -o bench bench.c
./bench
```

```text
checksum match: yes
malloc/free per object : 0.061 s
arena bump allocation  : 0.007 s
speedup                : 8.2x
```

Roughly 8x, and the second run gave 6.9x — the ratio moves with system
load, but the order of magnitude is stable. Two effects combine: the
per-call bookkeeping disappears, and the arena's objects come out
contiguous, so the summing loop walks straight through cache lines instead
of chasing pointers scattered across the heap.

The measurement matters more than the number. Never assume a custom
allocator is faster for *your* workload; modern `malloc` implementations are
extremely good, and a pool that thrashes cache can lose outright.

## The price: your allocator is invisible to the sanitizers

This is the trap that gets people, and it deserves its own demonstration.
The same logical bug — use after free — behaves completely differently
depending on who owns the memory:

```c
// blind.c -- the same bug, once through malloc and once through an arena
    /* variant 1 */
    char *p = malloc(32);
    strcpy(p, "malloc'd");
    free(p);
    printf("after free: %s\n", p);          /* use-after-free */

    /* variant 2 */
    static unsigned char arena[1024];
    size_t used = 0;
    char *p = (char *)(arena + used); used += 32;
    strcpy(p, "arena'd");
    used = 0;                                    /* "free" the whole arena */
    char *q = (char *)(arena + used); used += 32;
    strcpy(q, "REUSED");                         /* same bytes, new owner */
    printf("after reset: p says \"%s\"\n", p);   /* stale pointer, no complaint */
```

```bash
clang -Wall -Wextra -fsanitize=address -g -o blind blind.c
./blind m      # the malloc version
```

```text
==20844==ERROR: AddressSanitizer: heap-use-after-free on address 0x603000001c00
READ of size 2 at 0x603000001c00 thread T0
    #2 0x00010255c934 in main blind.c:11
```

```bash
./blind a      # the arena version
```

```text
after reset: p says "REUSED"
```

Identical bug class. AddressSanitizer catches the first instantly and says
nothing at all about the second, because as far as it is concerned that
memory is one live 1024-byte object that you are perfectly entitled to
write. You did not eliminate use-after-free; you eliminated the *detector*.

Mitigations, if you ship a custom allocator: bump a generation counter on
every reset and store it alongside handles instead of raw pointers;
`memset` freed pool blocks to `0xDD` in debug builds so stale reads produce
obviously wrong values; and wrap the allocator in ASan's manual poisoning
API (`ASAN_POISON_MEMORY_REGION`) so the sanitizer learns your object
boundaries.

## Cheat sheet

| Allocator | Alloc cost | Individual free? | Fits |
|---|---|---|---|
| `malloc`/`free` | High, general | Yes | Unknown sizes and lifetimes |
| Arena / bump | Pointer add | No — reset all at once | Per-request, per-frame, parse trees |
| Pool / free list | Pop a list | Yes, same-size only | Nodes, particles, fixed records |
| `realloc` growth | Amortised | n/a | Growable arrays — double, never `+1` |

| Tool | Purpose |
|---|---|
| `_Alignof(T)` / `alignof` | Required alignment of a type (C11) |
| `max_align_t` | Strictest alignment any scalar needs |
| `aligned_alloc(align, size)` | Heap block with alignment stronger than default (C11) |
| `(n + a - 1) & ~(a - 1)` | Round `n` up to a power-of-two multiple `a` |
| `offsetof(T, m)` | Byte offset of a member, for intrusive structures |
| `-fsanitize=address` | Catches `malloc` misuse — **not** custom-allocator misuse |
| `ASAN_POISON_MEMORY_REGION` | Teach ASan about your own allocator's free blocks |

Three more traps specific to hand-rolled allocators:

- Casting a `char *` in the middle of a byte array to `Rec *` and
  dereferencing it violates **strict aliasing** unless the alignment is
  right and the memory has no other declared type. Arenas backed by
  `malloc` are fine (that memory has no declared type); a `static unsigned
  char arena[]` is riskier, and `memcpy` is the portable escape hatch.
- Pointer arithmetic on `void *` is a GNU extension, not standard C. Cast to
  `unsigned char *` first — that is why every offset computation above does.
- A pointer into a reset arena is not merely stale in practice: reading it
  after the lifetime of the object it pointed to is undefined behaviour by
  the standard, so the optimiser is entitled to assume it never happens.

## Exercise

Build a **generation-tagged pool** that catches the use-after-free ASan
cannot see. Each block gets a `uint32_t generation` header; `pool_alloc`
returns a 64-bit *handle* packing the block index and the current
generation, and `pool_free` increments the block's generation. A `pool_deref`
function turns a handle back into a pointer, returning `NULL` when the
generations disagree.

Verify it three ways: allocate a block, free it, and confirm `pool_deref` on
the stale handle returns `NULL` rather than a pointer to reused data;
allocate/free the same slot 2^32 times' worth by seeding the counter near
wraparound and describe exactly what goes wrong; and benchmark
`pool_alloc` + `pool_deref` against plain `malloc`/`free` for the same
workload as `bench.c`. Report the speedup you actually measure, and be
honest if the validation ate it.
