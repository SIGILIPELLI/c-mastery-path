# 10 · Project — Multi-threaded Key-Value Store / Tiny HTTP Server

Everything in Level 3 converges here: a hash table with chaining
([module 01](01-data-structures.md)), heap discipline
([module 05](05-memory-management-deep-dive.md)), a multi-file build
([module 06](06-static-dynamic-libraries.md)), threads and locking
([module 07](07-concurrency-basics.md)), and sockets
([module 08](08-networking-basics.md)). The result is about 350 lines of C
that you can `curl` at from twenty terminals at once.

The design decision that shapes everything: the storage layer knows nothing
about HTTP, and the HTTP layer knows nothing about hash buckets. They meet
at four function signatures in `kv.h`.

## Layout

```text
kv/
├── kv.h        22 lines   public API -- the only thing server.c includes
├── kv.c       131 lines   hash table + reader/writer lock
├── server.c   128 lines   sockets, HTTP parsing, one thread per connection
├── kv_test.c   50 lines   correctness + a concurrency hammer
└── Makefile    23 lines   normal build and a sanitizer build
```

```c
// kv.h -- the whole contract
#ifndef KV_H
#define KV_H

#include <stddef.h>

typedef struct KvStore KvStore;      /* opaque: callers cannot touch fields */

KvStore *kv_create(size_t buckets);
void     kv_destroy(KvStore *s);

/* Returns 0 on success, -1 on allocation failure. Copies both strings. */
int      kv_set(KvStore *s, const char *key, const char *value);

/* Returns a malloc'd copy of the value, or NULL if absent. Caller frees. */
char    *kv_get(KvStore *s, const char *key);

/* Returns 1 if a key was removed, 0 if it was not present. */
int      kv_delete(KvStore *s, const char *key);

size_t   kv_count(KvStore *s);

#endif
```

`struct KvStore` is **declared but not defined** in the header — its fields
live only in `kv.c`. `server.c` can hold a `KvStore *` and pass it around,
but cannot reach past the lock into a bucket, which is exactly the property
that makes the locking auditable: every path into the table goes through one
of five functions in one file.

Two ownership rules are written into the comments because they are the ones
that leak or crash if broken: the store **copies** the strings you hand it
(so callers can pass stack buffers), and `kv_get` returns a **copy** the
caller must free.

## The storage layer

Buckets of singly-linked entries, FNV-1a hashing, and a
`pthread_rwlock_t` — the right lock here because a cache is read far more
often than written, and a read-write lock lets any number of readers proceed
in parallel while still excluding writers.

```c
// kv.c -- excerpt: the parts where the reasoning lives
struct KvStore {
    Entry **buckets;
    size_t nbuckets;
    size_t count;
    pthread_rwlock_t lock;
};

int kv_set(KvStore *s, const char *key, const char *value) {
    unsigned long idx = hash(key) % s->nbuckets;
    pthread_rwlock_wrlock(&s->lock);

    for (Entry *e = s->buckets[idx]; e; e = e->next) {
        if (strcmp(e->key, key) == 0) {
            char *nv = dup_str(value);
            if (!nv) { pthread_rwlock_unlock(&s->lock); return -1; }
            free(e->value);           /* only after the new copy succeeded */
            e->value = nv;
            pthread_rwlock_unlock(&s->lock);
            return 0;
        }
    }
    ...
}

char *kv_get(KvStore *s, const char *key) {
    unsigned long idx = hash(key) % s->nbuckets;
    pthread_rwlock_rdlock(&s->lock);
    char *out = NULL;
    for (Entry *e = s->buckets[idx]; e; e = e->next) {
        if (strcmp(e->key, key) == 0) { out = dup_str(e->value); break; }
    }
    pthread_rwlock_unlock(&s->lock);
    return out;                        /* a COPY: safe after the unlock */
}

int kv_delete(KvStore *s, const char *key) {
    unsigned long idx = hash(key) % s->nbuckets;
    pthread_rwlock_wrlock(&s->lock);
    Entry **link = &s->buckets[idx];   /* pointer-to-pointer: no special case
                                          for unlinking the head */
    while (*link) {
        Entry *e = *link;
        if (strcmp(e->key, key) == 0) {
            *link = e->next;
            free(e->key); free(e->value); free(e);
            s->count--;
            pthread_rwlock_unlock(&s->lock);
            return 1;
        }
        link = &e->next;
    }
    pthread_rwlock_unlock(&s->lock);
    return 0;
}
```

Three deliberate choices, each avoiding a specific bug:

- **`kv_get` returns a copy, not `e->value`.** Returning the internal
  pointer would hand the caller something another thread can `free` from
  `kv_set` or `kv_delete` a nanosecond after the lock is released — a
  use-after-free that only shows up under load.
- **`free(e->value)` happens after `dup_str` succeeds.** Freeing first and
  then failing to allocate leaves a dangling pointer in a live entry, and
  the store is now corrupt rather than merely out of memory.
- **`kv_delete` walks a `Entry **link`.** Unlinking through the address of
  the pointer that points at each node removes the usual "is it the head of
  the list?" special case entirely.

Every function takes the lock on entry and releases it on **every** return
path — including the error returns. One missed `pthread_rwlock_unlock` on a
rare allocation-failure branch deadlocks the whole server the first time
memory gets tight.

## The server layer

One thread per connection, detached so nobody has to join it:

```c
// server.c -- excerpt
static void *worker(void *arg) {
    int c = (int)(intptr_t)arg;
    handle(c);
    close(c);
    return NULL;
}

int main(void) {
    signal(SIGPIPE, SIG_IGN);          /* a client hanging up must not kill us */
    store = kv_create(64);
    ...
    for (;;) {
        int c = accept(srv, NULL, NULL);
        if (c < 0) continue;
        pthread_t t;
        if (pthread_create(&t, NULL, worker, (void *)(intptr_t)c) != 0) {
            close(c);                  /* refuse politely rather than leak */
            continue;
        }
        pthread_detach(t);             /* nobody will ever join this thread */
    }
}
```

The fd travels as `(void *)(intptr_t)c` rather than `&c` on purpose. Passing
`&c` would give every thread a pointer to the *same* loop variable that
`accept` keeps overwriting — the argument-lifetime trap from
[module 07](07-concurrency-basics.md), here made worse because two threads
would end up serving and closing the same socket.

Requests are read until the blank line that ends the headers, not with a
single `recv`:

```c
static ssize_t read_request(int c, char *buf, size_t cap) {
    size_t used = 0;
    while (used < cap - 1) {
        ssize_t n = recv(c, buf + used, cap - 1 - used, 0);
        if (n <= 0) break;
        used += (size_t)n;
        buf[used] = '\0';
        if (strstr(buf, "\r\n\r\n")) break;
    }
    buf[used] = '\0';
    return (ssize_t)used;
}
```

Routing is deliberately crude — the path is the key, and `PUT` takes the
value from a `?value=` query string:

```c
    char method[8] = "", path[512] = "";
    if (sscanf(buf, "%7s %511s", method, path) != 2) {
        respond(c, "400 Bad Request", "malformed request line\n");
        return;
    }
    const char *key = path + 1;                  /* skip the leading '/' */
```

Those `%7s`/`%511s` width limits are the difference between a toy and a
remote stack overflow. Every buffer here is fixed-size, every write into one
is bounded by `snprintf` or a width-limited `sscanf`, and the request buffer
is capped at 4 KB.

## Build

```makefile
CC      = cc
CFLAGS  = -Wall -Wextra -std=c11 -O2 -pthread
LDFLAGS = -pthread

all: kvserver kv_test

kvserver: server.o kv.o
	$(CC) $(LDFLAGS) -o $@ server.o kv.o

kv_test: kv_test.o kv.o
	$(CC) $(LDFLAGS) -o $@ kv_test.o kv.o

%.o: %.c kv.h
	$(CC) $(CFLAGS) -c $< -o $@

san: CFLAGS += -fsanitize=address,undefined -g -O1
san: LDFLAGS += -fsanitize=address,undefined
san: clean all

clean:
	rm -f *.o kvserver kv_test

.PHONY: all clean san
```

The `%.o: %.c kv.h` rule makes every object depend on the header, so
changing an API signature rebuilds both `.c` files instead of silently
linking a stale object against a new prototype.

```bash
make
```

```text
cc -Wall -Wextra -std=c11 -O2 -pthread -c server.c -o server.o
cc -Wall -Wextra -std=c11 -O2 -pthread -c kv.c -o kv.o
cc -pthread -o kvserver server.o kv.o
cc -Wall -Wextra -std=c11 -O2 -pthread -c kv_test.c -o kv_test.o
cc -pthread -o kv_test kv_test.o kv.o
```

## Running it

The storage layer is testable without a socket in sight — `kv_test` checks
the single-threaded semantics, then throws eight threads at the table:

```bash
./kv_test
```

```text
get lang           -> C
get lang (updated) -> C23
get missing        -> (null)
delete lang        -> 1
delete lang again  -> 0
count after        -> 0

8 threads x 2000 ops done, 344 keys left
store destroyed cleanly
```

Now the server:

```bash
./kvserver &
curl -s -X PUT "http://localhost:8081/lang?value=C"
curl -s -X PUT "http://localhost:8081/year?value=1972"
curl -s http://localhost:8081/lang
curl -s http://localhost:8081/
curl -s -i http://localhost:8081/missing
curl -s -X DELETE http://localhost:8081/year
curl -s http://localhost:8081/
```

```text
stored
stored
C
2 keys stored
HTTP/1.1 404 Not Found
Content-Type: text/plain
Content-Length: 12
deleted
1 keys stored
```

Then the part that actually exercises the locking — 200 writes from 20
concurrent clients, against a build compiled with
`-fsanitize=address,undefined`:

```bash
make san
./kvserver &
seq 1 200 | xargs -P 20 -I{} curl -s -o /dev/null -X PUT "http://localhost:8081/key{}?value=val{}"
curl -s http://localhost:8081/
curl -s http://localhost:8081/key137
```

```text
200 keys stored
val137
```

Every key survived, no value was interleaved with another, and the
sanitizers printed nothing — no overflow, no use-after-free, no undefined
behaviour on any of the 200 concurrent paths. Running the same test binary
under `-fsanitize=thread` is equally quiet:

```bash
cc -Wall -Wextra -std=c11 -g -fsanitize=thread -pthread -o kv_test_tsan kv_test.c kv.c
./kv_test_tsan
```

```text
8 threads x 2000 ops done, 344 keys left
store destroyed cleanly
```

Silence from ThreadSanitizer is the real result here. Delete a single
`pthread_rwlock_rdlock` from `kv_get` and it starts reporting a data race on
the very first run, even though the plain build would keep returning correct
answers for hours.

## Stretch goals

1. **Bounded memory.** Add `kv_set_max_entries()` and an LRU eviction
   policy: keep a doubly-linked list threaded through the entries, move an
   entry to the front on every `kv_get`, and evict from the back when the
   table is full. The hard part is that `kv_get` now *mutates* the list, so
   it can no longer take a read lock — measure how much throughput that
   costs you under `xargs -P 20`.

2. **Lock striping.** One global rwlock means every writer blocks every
   other writer across all 64 buckets. Replace it with an array of 16 locks,
   each guarding `bucket_index % 16`. Benchmark before and after with a
   write-heavy load; then explain why `kv_count` becomes the awkward
   operation, and pick between an atomic counter and taking all 16 locks.

3. **Persistence.** Append every `kv_set`/`kv_delete` to a write-ahead log
   with `write()` (not `fprintf` — you want control over when bytes hit the
   kernel), and replay it at startup. Add `fsync` on a `?durable=1` flag and
   measure the cost per request; the gap between "written" and "survives a
   power cut" is worth feeling in numbers.

4. **A real HTTP body.** Accept `PUT` values in the request body instead of
   the query string, which means parsing `Content-Length` and looping on
   `recv` until that many bytes past the header terminator have arrived.
   Test it with `curl -X PUT --data-binary @somefile`, and make sure a
   client that sends a `Content-Length` larger than it delivers eventually
   times out rather than pinning a thread forever.

5. **Stop spawning a thread per connection.** Replace it with a fixed pool
   of eight workers pulling accepted fds off a bounded queue guarded by a
   mutex and two condition variables — the producer/consumer structure from
   [module 07](07-concurrency-basics.md). Then hammer it with 5,000
   connections and compare peak memory against the thread-per-connection
   version.

6. **Graceful shutdown.** Handle `SIGINT` by setting a `volatile
   sig_atomic_t` flag, breaking the accept loop, and calling `kv_destroy`.
   Verify with `-fsanitize=address` that nothing leaks — and note why the
   handler may only touch a `volatile sig_atomic_t` and never call `printf`
   or `free`.
