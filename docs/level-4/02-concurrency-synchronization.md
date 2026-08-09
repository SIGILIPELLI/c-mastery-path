# 02 · Concurrency & Synchronization

[Level 3's concurrency module](../level-3/07-concurrency-basics.md) covered
threads, mutexes and condition variables — enough to write correct
concurrent code. This module is about the layer underneath: C11 atomics,
memory ordering, and the two hardware realities (cache lines and
compare-and-swap) that decide whether your correct code is also fast.

It also covers a more uncomfortable idea. A racy program is not merely a
program that *might* produce the wrong answer. It is a program with
**undefined behaviour**, which means the optimiser is allowed to transform
it in ways that make the race disappear, reappear, or change shape between
builds. The first example demonstrates exactly that.

## Three ways to count, measured

```c
// counters.c -- racy vs mutex vs atomic, same workload, timed
#include <stdatomic.h>
#include <pthread.h>

#define THREADS 8
#define PER     1000000

static long            plain = 0;
static long            guarded = 0;
static atomic_long     atom = 0;
static pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;

static void *racy(void *_)   { (void)_; for (int i=0;i<PER;i++) plain++; return NULL; }

static void *locked(void *_) {
    (void)_;
    for (int i=0;i<PER;i++) { pthread_mutex_lock(&mtx); guarded++; pthread_mutex_unlock(&mtx); }
    return NULL;
}

static void *atomicf(void *_) {
    (void)_;
    for (int i=0;i<PER;i++) atomic_fetch_add_explicit(&atom, 1, memory_order_relaxed);
    return NULL;
}
```

Built at `-O0`, eight threads, a million increments each:

```text
expected      8000000

plain            1829261   0.006 s  WRONG
mutex            8000000   0.248 s  ok
atomic           8000000   0.279 s  ok
```

The racy version lost 77% of its increments. Now the *same source* at `-O2`:

```text
expected      8000000

plain            7000000   0.000 s  WRONG
mutex            8000000   0.236 s  ok
atomic           8000000   0.159 s  ok
```

Look at what happened to `plain`: it lost exactly **1,000,000** — one
thread's entire contribution — in **0.000 seconds**. The optimiser noticed
that `plain` is not atomic and not `volatile`, concluded (legally, since a
data race is UB) that no other thread could be touching it, and collapsed
the million-iteration loop into a single `plain += 1000000`. Eight threads
then raced on one addition each, and one of the eight was lost.

That is the lesson worth carrying: at `-O0` the bug looked like "we lose
some increments." At `-O2` it became a different bug with a different
magnitude and a 40x speed change. You cannot debug a data race by reasoning
about what the machine does, because the compiler rewrote what the machine
does. Fix the race; do not characterise it.

The mutex and the atomic are both correct at both optimisation levels. At
`-O2` the atomic ran 1.5x faster than the mutex — a single `LDADD`
instruction versus a lock acquire, a store, and a release.

## Memory ordering

`atomic_fetch_add(&x, 1)` defaults to `memory_order_seq_cst`: the strongest
ordering, where every thread agrees on a single global order of all
sequentially-consistent operations. It is the safe default and the slow one.

The explicit forms let you ask for less:

| Order | Guarantees | Use for |
|---|---|---|
| `memory_order_relaxed` | Atomicity only — no ordering with other variables | Counters and statistics nobody reads until the end |
| `memory_order_acquire` | On a load: later reads/writes cannot move before it | Taking a lock, reading a published pointer |
| `memory_order_release` | On a store: earlier reads/writes cannot move after it | Releasing a lock, publishing a pointer |
| `memory_order_acq_rel` | Both, for read-modify-write ops | CAS in a lock-free structure |
| `memory_order_seq_cst` | Single total order across all threads (default) | When you are unsure — always correct |

The counter above uses `relaxed` because nothing else depends on *when*
each increment becomes visible; only the total after `pthread_join` matters,
and joining is itself a synchronisation point. Publishing a pointer to an
initialised object is the opposite case: the store must be `release` and the
load `acquire`, or another thread can legally see the pointer before the
bytes it points at.

On x86 the difference between `relaxed` and `seq_cst` loads is often
literally zero instructions, so a wrong ordering can pass every test you run
and then fail on ARM. Choose orderings by reasoning, never by benchmarking
on one architecture.

## Compare-and-swap: the primitive underneath everything

`atomic_compare_exchange_weak(&obj, &expected, desired)` atomically checks
whether `obj` still equals `expected`, and if so stores `desired`. If not,
it writes the *current* value back into `expected` and returns false. That
one instruction is enough to build lock-free data structures:

```c
// casstack.c -- lock-free push with compare-and-swap
typedef struct Node { int value; struct Node *next; } Node;

static _Atomic(Node *) top = NULL;

static void push(int v) {
    Node *n = malloc(sizeof *n);
    n->value = v;
    n->next = atomic_load_explicit(&top, memory_order_relaxed);
    /* Retry until nobody changed `top` between our read and our write.
       On failure the CAS reloads n->next with the current top for us. */
    while (!atomic_compare_exchange_weak_explicit(
               &top, &n->next, n,
               memory_order_release, memory_order_relaxed))
        ;
    atomic_fetch_add_explicit(&pushed, 1, memory_order_relaxed);
}
```

Eight threads pushing 50,000 nodes each, drained single-threaded after
joining:

```bash
clang -Wall -Wextra -std=c11 -O2 -pthread -o casstack casstack.c
./casstack
```

```text
threads pushed : 400000
nodes drained  : 400000
checksum       : 79999800000 (expected 79999800000) ok
atomic_int lock-free? yes, hardware CAS
```

Identical across runs, and silent under `-fsanitize=thread`. The `release`
ordering on success is what makes it safe: it guarantees that `n->value` is
written *before* `n` becomes reachable via `top`, so no thread can observe a
node whose contents have not landed yet.

Three details in that loop:

- The CAS writes the current value into `expected` on failure, and here
  `expected` **is** `n->next`. So a failed attempt automatically re-points
  the new node at the new top — the retry needs no body at all.
- `_weak` may fail spuriously even when the values match, which is why it
  must live in a loop. `_strong` does not, but is slower on architectures
  like ARM that implement CAS with load-linked/store-conditional. In a
  retry loop, always use `_weak`.
- `atomic_is_lock_free` tells you whether the compiler emitted a real
  hardware instruction or silently fell back to a hidden global mutex. A
  "lock-free" structure built on emulated atomics is slower than the mutex
  you were avoiding.

**Concurrent `pop` is much harder than `push`, and the difference is not
obvious.** A popping thread reads `top`, then reads `old->next` to build its
CAS — but between those two instructions another thread can pop and `free`
that same node, so `old->next` is a use-after-free. Worse, the node's
address can be recycled by `malloc` and pushed back, so the CAS *succeeds*
against a pointer that is no longer the same object. That is the **ABA
problem**, and solving it needs hazard pointers, epoch-based reclamation, or
a tagged pointer with a version counter. This is why the example above
pushes concurrently and drains after `pthread_join`: memory reclamation, not
the CAS, is the hard part of lock-free programming.

## False sharing: correct, atomic, and 48x too slow

Cache coherence works on **cache lines**, not variables. Two threads
updating two different counters that happen to sit in the same line will
bounce that line between cores on every write, even though they never touch
the same bytes.

```c
// falseshare.c -- same work, two memory layouts, one cache line apart
#define LINE 128           /* cache line size on this machine */

typedef struct { atomic_long v; } Packed;
typedef struct { atomic_long v; char pad[LINE - sizeof(atomic_long)]; } Padded;

static Packed packed[THREADS];
static Padded padded[THREADS];
```

Each of eight threads increments *its own* counter five million times:

```text
sizeof(Packed)=8  sizeof(Padded)=128
8 counters sharing cache lines : 0.925 s
8 counters on separate lines   : 0.019 s
padding won by                 : 48.2x
```

Forty-eight times, from adding padding. Both versions are correct; both use
the same atomic operation the same number of times; no thread ever reads
another's counter. In the packed version all eight counters fit in one or
two 128-byte lines, so every increment invalidates the line in seven other
cores' caches.

This is the standard reason a program "doesn't scale" — throughput gets
*worse* as you add threads. The fix is layout, not synchronisation: pad
per-thread state to a cache line, or better, keep each thread's accumulator
in a **local** variable and merge once at the end, which needs no atomics at
all.

## Cheat sheet

| Construct | Purpose |
|---|---|
| `#include <stdatomic.h>` | C11 atomics (check `__STDC_NO_ATOMICS__`) |
| `atomic_int`, `atomic_long`, `_Atomic(T *)` | Atomic types |
| `atomic_load` / `atomic_store` | Race-free read / write |
| `atomic_fetch_add` / `_sub` / `_or` | Atomic read-modify-write, returns the old value |
| `atomic_exchange(&x, v)` | Swap, returns the old value |
| `atomic_compare_exchange_weak(&x, &exp, des)` | CAS; retry in a loop |
| `atomic_flag` + `ATOMIC_FLAG_INIT` | The only type guaranteed lock-free everywhere |
| `atomic_is_lock_free(&x)` | Is this a real instruction or an emulated lock? |
| `atomic_thread_fence(order)` | Standalone barrier, no associated object |
| `pthread_rwlock_t` | Many readers or one writer — for read-heavy shared state |
| `pthread_once` / `call_once` | Run an initialiser exactly once |
| `_Thread_local` | Per-thread storage, no synchronisation needed |
| `-fsanitize=thread` | Detect races that did not happen to manifest |

Traps specific to this level:

- **`volatile` is not atomic.** It prevents the compiler from caching a
  value in a register; it provides no atomicity and no ordering. It is for
  memory-mapped hardware and `sig_atomic_t` signal flags, never for threads.
- **Atomic on the wrong granularity.** `atomic_load(&head)` followed by
  `head->next` is two operations; the object can change in between. Atomicity
  applies to one access, not to a sequence.
- **A `long` is not atomic just because it fits in a register.** The
  compiler may split, duplicate, or eliminate a non-atomic access entirely,
  as `-O2` did above.
- **`errno`, `strtok`, `rand`, `localtime`** — check thread safety before
  calling anything from a thread. `errno` is per-thread; `strtok` and `rand`
  are not (`strtok_r`, `rand_r` are).

## Exercise

Take the packed/padded benchmark and add a **third** variant: each thread
accumulates into a plain `long` local variable and does one
`atomic_fetch_add` into the shared counter at the end. Predict where it will
land relative to the other two, then measure it, and explain the result you
actually get.

Then make the false sharing appear and disappear on demand. Replace the
fixed `LINE` with a compile-time `-DPAD=n` and sweep `n` from 8 to 256 in
powers of two, plotting time against padding size. The step change tells you
your machine's real cache line size — compare it against
`sysctl hw.cachelinesize` (macOS) or
`/sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size` (Linux).

Finally, build the counters benchmark at `-O0`, `-O1`, `-O2` and `-O3` and
record the racy result at each level. Then inspect the generated assembly
with `clang -S -O2` and find the instruction that replaced the loop — the
concrete evidence that a data race is a licence for the compiler to rewrite
your program, not just a scheduling accident.
