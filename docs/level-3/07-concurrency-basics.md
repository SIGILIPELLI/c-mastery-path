# 07 · Concurrency Basics

Every program so far in this path has done one thing at a time. A
**thread** is a second (third, fourth…) flow of execution inside the *same*
process, sharing the same heap, the same globals, and the same open file
descriptors. That sharing is the whole point — and the whole danger. This
module covers POSIX threads (pthreads): creating them, getting results back,
and the locking you need the moment two of them touch the same memory.

Everything here compiles with `-pthread`, which both defines the right
feature macros and links the threading library:

```bash
gcc -Wall -Wextra -pthread -o prog prog.c
```

## Creating and joining threads

`pthread_create` starts a thread running a function with the signature
`void *f(void *)`. `pthread_join` blocks until that thread finishes.

```c
// hello_threads.c -- create threads, pass each its own argument, join them
#include <stdio.h>
#include <pthread.h>

void *worker(void *arg) {
    int id = *(int *)arg;
    printf("worker %d running\n", id);
    return NULL;
}

int main(void) {
    pthread_t threads[3];
    int ids[3];

    for (int i = 0; i < 3; i++) {
        ids[i] = i;
        if (pthread_create(&threads[i], NULL, worker, &ids[i]) != 0) {
            perror("pthread_create");
            return 1;
        }
    }

    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("all workers finished\n");
    return 0;
}
```

```text
worker 0 running
worker 2 running
worker 1 running
all workers finished
```

Note the order: worker 2 printed before worker 1. **Thread scheduling is
not deterministic** — run this again and you may well get a different
order. Any program whose correctness depends on which thread happens to run
first is already broken; it just hasn't failed yet.

Two traps live in that argument-passing code:

- Each thread gets `&ids[i]` — a *distinct* address. The tempting
  `pthread_create(..., &i)` passes every thread a pointer to the **same**
  loop variable, which `main` keeps incrementing while the threads read it.
  You get whatever value `i` holds at the moment each thread dereferences,
  which is usually not what you wanted.
- `ids` must outlive the threads. Pointing a thread at a local in a function
  that returns before `pthread_join` is a dangling pointer into a dead stack
  frame.

If you forget `pthread_join` entirely, `main` returns, the process exits,
and unfinished threads are killed mid-work with no warning.

## Getting results back

The cleanest way to return data is to hand each thread a pointer to a struct
it owns exclusively, and read the results after joining:

```c
// sum_threads.c -- each thread sums a slice and returns its result
#include <stdio.h>
#include <pthread.h>

typedef struct {
    const int *data;
    int start, end;      // half-open range [start, end)
    long sum;            // filled in by the thread
} Slice;

void *sum_slice(void *arg) {
    Slice *s = arg;
    s->sum = 0;
    for (int i = s->start; i < s->end; i++)
        s->sum += s->data[i];
    return NULL;
}

int main(void) {
    enum { N = 1000, T = 4 };
    int data[N];
    for (int i = 0; i < N; i++) data[i] = i + 1;   // 1..1000

    pthread_t th[T];
    Slice slices[T];
    int chunk = N / T;

    for (int i = 0; i < T; i++) {
        slices[i] = (Slice){ .data = data,
                             .start = i * chunk,
                             .end   = (i == T - 1) ? N : (i + 1) * chunk };
        pthread_create(&th[i], NULL, sum_slice, &slices[i]);
    }

    long total = 0;
    for (int i = 0; i < T; i++) {
        pthread_join(th[i], NULL);
        printf("slice %d [%3d,%4d) = %ld\n",
               i, slices[i].start, slices[i].end, slices[i].sum);
        total += slices[i].sum;          // main thread adds, so no lock needed
    }
    printf("total = %ld\n", total);
    return 0;
}
```

```text
slice 0 [  0, 250) = 31375
slice 1 [250, 500) = 93875
slice 2 [500, 750) = 156375
slice 3 [750,1000) = 218875
total = 500500
```

No locks appear anywhere, yet this is completely safe: every thread writes
to a *different* `Slice`, and `main` only reads `slices[i].sum` **after**
`pthread_join(th[i], NULL)` — a synchronization point that guarantees that
thread's writes are visible. Partitioning data so threads never touch the
same bytes is always cheaper and safer than locking; reach for it first.

## The race condition, demonstrated

Now let threads share one variable:

```c
// race.c -- four threads increment one shared counter with no protection
#include <stdio.h>
#include <pthread.h>

#define THREADS    4
#define PER_THREAD 200000

long counter = 0;   // shared by every thread

void *bump(void *arg) {
    (void)arg;
    for (int i = 0; i < PER_THREAD; i++) {
        counter++;          // read, add, write -- three steps, not one
    }
    return NULL;
}

int main(void) {
    pthread_t t[THREADS];

    for (int i = 0; i < THREADS; i++)
        pthread_create(&t[i], NULL, bump, NULL);
    for (int i = 0; i < THREADS; i++)
        pthread_join(t[i], NULL);

    printf("expected: %d\n", THREADS * PER_THREAD);
    printf("actual:   %ld\n", counter);
    return 0;
}
```

Three consecutive runs of the *same* binary:

```text
expected: 800000     expected: 800000     expected: 800000
actual:   212897     actual:   267208     actual:   217715
```

`counter++` is not one instruction. It is *load the value, add one, store it
back*. When two threads load `500` at the same moment, both compute `501`,
both store `501`, and one increment vanishes. Roughly three quarters of the
work here disappeared — and the program never crashed, never warned, and
exited with status 0. That is why concurrency bugs are so expensive: the
failure mode is a quietly wrong number, discovered days later.

**ThreadSanitizer** finds these deterministically, without needing the bug
to actually manifest on that particular run:

```bash
gcc -Wall -Wextra -fsanitize=thread -g -o race_tsan race.c
./race_tsan
```

```text
WARNING: ThreadSanitizer: data race (pid=7157)
  Write of size 8 at 0x000100044000 by thread T2:
    #0 bump race.c:13

  Previous write of size 8 at 0x000100044000 by thread T1:
    #0 bump race.c:13

  Location is global 'counter' at 0x000100044000
SUMMARY: ThreadSanitizer: data race race.c:13 in bump
```

It names the variable, the line, and both threads involved. Run every
threaded program under `-fsanitize=thread` at least once.

## Mutexes: one thread in the critical section at a time

A **mutex** (mutual exclusion lock) makes a block of code indivisible: at
most one thread holds the lock, and everyone else blocks inside
`pthread_mutex_lock` until it is released.

```c
// mutex.c -- the same counter, protected by a mutex
#include <stdio.h>
#include <pthread.h>

#define THREADS    4
#define PER_THREAD 200000

long counter = 0;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

void *bump(void *arg) {
    (void)arg;
    for (int i = 0; i < PER_THREAD; i++) {
        pthread_mutex_lock(&lock);
        counter++;                      // critical section
        pthread_mutex_unlock(&lock);
    }
    return NULL;
}

int main(void) {
    pthread_t t[THREADS];

    for (int i = 0; i < THREADS; i++)
        pthread_create(&t[i], NULL, bump, NULL);
    for (int i = 0; i < THREADS; i++)
        pthread_join(t[i], NULL);

    printf("expected: %d\n", THREADS * PER_THREAD);
    printf("actual:   %ld\n", counter);

    pthread_mutex_destroy(&lock);
    return 0;
}
```

```text
expected: 800000
actual:   800000
```

Correct, and correct on every run. The rules that keep it that way:

- **Every** access to the shared data must take the lock, reads included. One
  unlocked read is enough to reintroduce the race.
- Hold the lock for as short a span as possible. Locking around a `printf`
  or a file write serializes your threads back into a slow single-threaded
  program.
- Never `return` (or `goto`, or `break` to an outer scope) out of a critical
  section without unlocking — the next thread waits forever.
- Two mutexes taken in opposite orders by two threads is a **deadlock**:
  thread A holds `x` wanting `y`, thread B holds `y` wanting `x`, both stop
  permanently. Fix it by always acquiring locks in one globally agreed order.

## Condition variables: waiting for state to change

A mutex protects data; it does not let a thread *wait* for that data to
reach some condition. Spinning on `while (count == 0) {}` burns a whole CPU
core. A **condition variable** lets a thread sleep until another thread
signals it.

```c
// queue.c -- one producer, two consumers, sharing a bounded queue
#include <stdio.h>
#include <pthread.h>

#define CAP   4
#define ITEMS 8

int buf[CAP];
int count = 0, head = 0, tail = 0;
int done = 0;

pthread_mutex_t m         = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t  not_full  = PTHREAD_COND_INITIALIZER;
pthread_cond_t  not_empty = PTHREAD_COND_INITIALIZER;

void *producer(void *arg) {
    (void)arg;
    for (int i = 1; i <= ITEMS; i++) {
        pthread_mutex_lock(&m);
        while (count == CAP)                        // while, not if
            pthread_cond_wait(&not_full, &m);
        buf[tail] = i;
        tail = (tail + 1) % CAP;
        count++;
        printf("produced %d (queue depth %d)\n", i, count);
        pthread_cond_signal(&not_empty);
        pthread_mutex_unlock(&m);
    }
    pthread_mutex_lock(&m);
    done = 1;
    pthread_cond_broadcast(&not_empty);             // wake every waiter
    pthread_mutex_unlock(&m);
    return NULL;
}

void *consumer(void *arg) {
    long id = (long)arg;
    for (;;) {
        pthread_mutex_lock(&m);
        while (count == 0 && !done)
            pthread_cond_wait(&not_empty, &m);
        if (count == 0 && done) {                   // drained and finished
            pthread_mutex_unlock(&m);
            return NULL;
        }
        int item = buf[head];
        head = (head + 1) % CAP;
        count--;
        pthread_cond_signal(&not_full);
        pthread_mutex_unlock(&m);

        printf("consumer %ld got %d\n", id, item);  // work outside the lock
    }
}

int main(void) {
    pthread_t p, c1, c2;
    pthread_create(&p,  NULL, producer, NULL);
    pthread_create(&c1, NULL, consumer, (void *)1L);
    pthread_create(&c2, NULL, consumer, (void *)2L);

    pthread_join(p, NULL);
    pthread_join(c1, NULL);
    pthread_join(c2, NULL);
    printf("queue drained\n");
    return 0;
}
```

```text
produced 1 (queue depth 1)
produced 2 (queue depth 2)
produced 3 (queue depth 3)
produced 4 (queue depth 4)
consumer 1 got 1
consumer 1 got 2
consumer 1 got 3
consumer 1 got 4
produced 5 (queue depth 1)
...
consumer 1 got 8
queue drained
```

The producer filled the queue to its cap of 4, then blocked in
`pthread_cond_wait(&not_full, &m)` until a consumer made room — the ring
buffer never overflowed even though nothing coordinates the two loops
explicitly.

Three details do all the work:

1. `pthread_cond_wait` **atomically releases the mutex and sleeps**, then
   re-acquires it before returning. That atomicity is why you must already
   hold the lock when you call it: otherwise a signal can slip in between
   your check and your sleep, and you wait for an event that already fired.
2. The wait sits inside `while`, never `if`. A thread can wake without
   anyone signalling it (a *spurious wakeup*), and another thread may have
   taken the item first anyway. Re-check the condition after every wake.
3. `signal` wakes one waiter; `broadcast` wakes all. Shutdown uses
   `broadcast` because *both* consumers must see `done` and exit — a single
   `signal` would leave one blocked forever, and `pthread_join` would hang.

Also notice which line sits *outside* the lock: the `printf` of the consumed
item. Once the item has been copied out of the buffer, no shared state is
involved, so holding the mutex across the slow I/O call would only stop
other threads from making progress.

## Cheat sheet

| Call | Purpose |
|---|---|
| `pthread_create(&tid, NULL, fn, arg)` | Start a thread running `fn(arg)`; returns 0 on success |
| `pthread_join(tid, &ret)` | Block until the thread ends; also a memory-visibility barrier |
| `pthread_detach(tid)` | Never join this one; its resources are freed automatically |
| `pthread_self()` | The calling thread's own id |
| `pthread_mutex_lock` / `_unlock` | Enter / leave a critical section |
| `pthread_mutex_trylock` | Take the lock only if free; returns `EBUSY` instead of blocking |
| `pthread_cond_wait(&cv, &m)` | Release `m`, sleep, re-acquire `m` on wake |
| `pthread_cond_signal` / `_broadcast` | Wake one waiter / all waiters |
| `PTHREAD_MUTEX_INITIALIZER` | Static initializer for a global mutex |
| `-pthread` | Compile *and* link flag; required |
| `-fsanitize=thread` | Detect data races at runtime |

## Exercise

Turn `sum_threads.c` into a **parallel word counter**. Give each thread a
slice of a large text buffer, have it count words in its slice, and combine
the results two ways:

1. Each thread writes its count into its own struct field, and `main` totals
   them after joining (no lock anywhere).
2. Each thread adds directly into a single shared `long total_words` — first
   without a mutex, then with one.

Run version 2's unlocked build about ten times and record how often the
total is wrong, then confirm `-fsanitize=thread` flags it on the *first* run
even when the arithmetic happened to come out right. Finally, time all three
versions on a buffer large enough to take a second or so, and explain why
the mutex version can be slower than the lock-free partitioned one even
though both produce the correct answer.
