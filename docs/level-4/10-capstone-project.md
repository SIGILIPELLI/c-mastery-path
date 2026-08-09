# 10 · Capstone Project

**tinydb** — an in-memory database engine with a command language, a
write-ahead log for crash recovery, a test suite, two build systems, and CI.
About 560 lines across nine files.

Every module in this path shows up somewhere: hash tables and intrusive
lists, opaque handles and ownership rules, bounded string handling, error
codes instead of exceptions, `strtoll` instead of `atoi`, dependency-tracked
builds, sanitizers wired into the test target, and a coverage number that
tells you where the tests are thin rather than a green tick that says
nothing.

## Layout

```text
tinydb/
├── include/
│   ├── db.h                39   public storage API (opaque Db)
│   └── parse.h             19   command representation
├── src/
│   ├── db.c               149   hash index + order list + WAL
│   ├── parse.c             71   tokenizer and validating parser
│   └── main.c              52   REPL, the only file that does I/O
├── tests/
│   └── test_db.c          131   36 assertions across 6 test functions
├── Makefile                37   incremental build, check, asan targets
├── CMakeLists.txt          27   the same project for CMake/CTest
└── .github/workflows/ci.yml 37  4-way matrix + sanitizers + coverage
```

The rule that shapes it: **`db.c` knows nothing about text, `parse.c` knows
nothing about storage, and only `main.c` performs I/O.** That is what makes
`test_db.c` able to test the entire engine without a terminal, a file, or a
socket.

## The public API

```c
// include/db.h
typedef struct Db Db;          /* opaque: callers cannot reach the fields */

typedef struct {
    int64_t id;
    char    name[DB_MAX_NAME];
    int64_t score;
} Row;

typedef enum {
    DB_OK = 0, DB_ERR_NOMEM, DB_ERR_DUPLICATE, DB_ERR_NOT_FOUND, DB_ERR_IO
} DbStatus;

const char *db_strerror(DbStatus s);

Db      *db_open(const char *wal_path);   /* wal_path may be NULL: memory only */
void     db_close(Db *db);

DbStatus db_insert(Db *db, int64_t id, const char *name, int64_t score);
DbStatus db_delete(Db *db, int64_t id);
const Row *db_get(const Db *db, int64_t id);
size_t   db_count(const Db *db);

/* Visit rows with score >= min_score, in insertion order. Returns count. */
typedef void (*RowVisitor)(const Row *r, void *ctx);
size_t   db_scan(const Db *db, int64_t min_score, RowVisitor fn, void *ctx);
```

Four deliberate API decisions:

- **`DbStatus`, not `-1`.** A named enum with `db_strerror` means the caller
  can distinguish "duplicate" from "out of memory" and report something
  useful. `DB_OK = 0` keeps `if (s != DB_OK)` idiomatic.
- **`db_get` returns `const Row *`, borrowed.** No allocation, no ownership
  transfer, no `free` for the caller to forget — and `const` documents that
  the pointer is a view, invalidated by the next mutation. (Contrast the
  [Level 3 KV store](../level-3/10-project-kv-store.md), which returns a
  copy because it is accessed concurrently.)
- **`db_scan` takes a callback plus a `void *ctx`.** The `ctx` parameter is
  what separates a usable C callback from a useless one; without it the
  visitor can only touch globals.
- **`int64_t` and `size_t`, never `int` or `long`.** Widths are pinned, per
  [module 04](04-writing-portable-c.md).

## Storage: two chains through one node

Rows need O(1) lookup by id *and* stable insertion order for `SELECT`. Both
come from one allocation with two `next` pointers:

```c
// src/db.c
typedef struct Entry {
    Row row;
    struct Entry *hnext;      /* hash chain */
    struct Entry *onext;      /* insertion order chain */
    int alive;
} Entry;

struct Db {
    Entry *buckets[BUCKETS];
    Entry *order_head, *order_tail;
    size_t count;
    FILE  *wal;
};

static size_t bucket_of(int64_t id) {
    uint64_t h = (uint64_t)id * 0x9E3779B97F4A7C15ULL;   /* Fibonacci hashing */
    return (size_t)(h >> 56) & (BUCKETS - 1);
}
```

Sequential ids (1, 2, 3, …) would land in consecutive buckets under a plain
`% BUCKETS`, which is fine — but multiplying by the 64-bit golden ratio and
taking the *high* bits scatters any pattern, including the ids-that-differ-
by-256 case that would otherwise collide every time. Taking the high bits
matters: the low bits of a multiplication are the least mixed.

Deletion uses a **tombstone** rather than unlinking:

```c
static DbStatus apply_delete(Db *db, int64_t id) {
    Entry *e = find(db, id);
    if (!e) return DB_ERR_NOT_FOUND;
    e->alive = 0;              /* tombstone: order list stays intact */
    db->count--;
    return DB_OK;
}
```

Unlinking from a singly-linked order list requires finding the predecessor —
O(n) — and freeing the node would invalidate any borrowed `const Row *` the
caller still holds. The tombstone makes delete O(1) and keeps the borrowing
contract honest. The cost is that memory is only reclaimed at `db_close`,
which is exactly the kind of trade a real engine makes explicit (and then
fixes with compaction — see the stretch goals).

## Crash recovery: the same function for writes and replay

The write-ahead log is what makes this a database rather than a hash table.
The key structural choice is that mutations are split into an `apply_*`
function that changes memory and a public function that logs first:

```c
/* Apply without logging -- used both by db_insert and by WAL replay. */
static DbStatus apply_insert(Db *db, int64_t id, const char *name, int64_t score) { ... }

DbStatus db_insert(Db *db, int64_t id, const char *name, int64_t score) {
    DbStatus s = apply_insert(db, id, name, score);
    if (s != DB_OK) return s;
    if (db->wal) {
        if (fprintf(db->wal, "INS %lld %s %lld\n",
                    (long long)id, db->order_tail->row.name, (long long)score) < 0)
            return DB_ERR_IO;
        fflush(db->wal);
    }
    return DB_OK;
}
```

Recovery then *is* the write path, minus the logging:

```c
static void replay(Db *db) {
    if (!db->wal) return;
    rewind(db->wal);
    ...
    while (fgets(line, sizeof line, db->wal)) {
        if (sscanf(line, "%7s %lld %31s %lld", op, &id, name, &score) == 4 &&
            strcmp(op, "INS") == 0) {
            apply_insert(db, id, name, score);
        } else if (sscanf(line, "%7s %lld", op, &id) == 2 &&
                   strcmp(op, "DEL") == 0) {
            apply_delete(db, id);
        }
    }
    fseek(db->wal, 0, SEEK_END);
}
```

One code path for both means recovery cannot drift out of sync with normal
operation — the classic way real systems corrupt data. Note the `%7s` and
`%31s` widths (bounded by the destination sizes) and that the log records
`db->order_tail->row.name`, the *truncated* name actually stored, so replay
reproduces the stored state rather than the input.

`fflush` after each record makes the write visible to the OS. It does **not**
survive a power cut — that needs `fsync`, and the difference is a stretch
goal below.

## Parsing: reject early, validate everything

```c
// src/parse.c
/* strtoll with full error checking -- never atoi. */
static int parse_i64(const char *s, int64_t *out) {
    if (!s || !*s) return 0;
    errno = 0;
    char *end;
    long long v = strtoll(s, &end, 10);
    if (errno == ERANGE || *end != '\0' || end == s) return 0;
    *out = (int64_t)v;
    return 1;
}
```

Three checks that `atoi` cannot make: `ERANGE` catches overflow, `*end`
catches trailing garbage (`"12abc"`), and `end == s` catches input with no
digits at all. `errno` is cleared first because it is only meaningful after
a failure ([Level 3 module 09](../level-3/09-unix-system-calls.md)).

The parser returns a `Command` **by value** with an embedded error string,
so there is nothing to free on the error path and no ownership question for
the caller.

## Build

```bash
make
```

```text
cc -std=c11 -Wall -Wextra -Werror -O2 -Iinclude -D_POSIX_C_SOURCE=200809L -MMD -MP -c src/db.c -o build/src/db.o
cc -std=c11 -Wall -Wextra -Werror -O2 -Iinclude -D_POSIX_C_SOURCE=200809L -MMD -MP -c src/main.c -o build/src/main.o
cc -std=c11 -Wall -Wextra -Werror -O2 -Iinclude -D_POSIX_C_SOURCE=200809L -MMD -MP -c src/parse.c -o build/src/parse.o
cc  -o build/tinydb build/src/db.o build/src/main.o build/src/parse.o
```

`-Werror` is on from the first commit. Warnings that are merely printed get
scrolled past; warnings that stop the build get fixed.

CMake builds the same project for CTest and IDEs:

```bash
cmake -S . -B cmake-build -DCMAKE_BUILD_TYPE=Release && cmake --build cmake-build
cd cmake-build && ctest --output-on-failure
```

```text
1/1 Test #1: unit .............................   Passed    0.52 sec
100% tests passed out of 1
```

## Running it

Tests first — including crash recovery, which opens a real WAL, closes the
handle to simulate a restart, and reopens it:

```bash
make check
```

```text
insert_and_get         ok
duplicate_and_delete   ok
scan_filters           ok
long_name_truncates    ok
parser                 ok
wal_recovery           ok

36 checks, 0 failed
```

The same suite, rebuilt with AddressSanitizer and UndefinedBehaviorSanitizer:

```bash
make asan
```

```text
insert_and_get         ok
duplicate_and_delete   ok
scan_filters           ok
long_name_truncates    ok
parser                 ok
wal_recovery           ok

36 checks, 0 failed
```

Silence from the sanitizers across every allocation, tombstone, truncated
name and WAL replay is the result that matters.

Now the engine itself. Session one:

```bash
printf 'INSERT 1 alice 50\nINSERT 2 bob 90\nINSERT 3 carol 70\nINSERT 1 dup 1\nDELETE 2\nSELECT\nSELECT 60\nCOUNT\nDROP TABLE users\nINSERT abc x 5\nQUIT\n' | ./build/tinydb demo.log
```

```text
opened demo.log (0 rows recovered)
inserted
inserted
inserted
duplicate id
deleted
  1      alice            50
  3      carol            70
(2 rows)
  3      carol            70
(1 rows)
2
error: unknown command (INSERT/DELETE/SELECT/COUNT/QUIT)
error: bad id
```

The duplicate was rejected, the filter worked, and both malformed commands
produced a diagnostic instead of a crash or a wrong answer.

The log on disk:

```bash
cat demo.log
```

```text
INS 1 alice 50
INS 2 bob 90
INS 3 carol 70
DEL 2
```

Session two — a fresh process, no shared memory, recovering from that file
alone:

```bash
printf 'COUNT\nSELECT\nQUIT\n' | ./build/tinydb demo.log
```

```text
opened demo.log (2 rows recovered)
2
  1      alice            50
  3      carol            70
(2 rows)
```

Three inserts and a delete replayed exactly, including the deletion. That is
durability.

## Coverage: where the tests are thin

```bash
clang -std=c11 -Iinclude -D_POSIX_C_SOURCE=200809L \
      -fprofile-instr-generate -fcoverage-mapping -g \
      -o test_cov tests/test_db.c src/db.c src/parse.c
LLVM_PROFILE_FILE=t.profraw ./test_cov
llvm-profdata merge -sparse t.profraw -o t.profdata
llvm-cov report ./test_cov -instr-profile=t.profdata src/
```

```text
Filename         Lines      Missed Lines     Cover    Branches   Missed Branches     Cover
parse.c             52                 5    90.38%          49                21    57.14%
db.c               108                11    89.81%          69                23    66.67%
TOTAL              160                16    90.00%         118                44    62.71%
```

90% of lines but only **63% of branches**, and that gap is the honest
summary of this test suite. The uncovered branches are mostly allocation
failures (`calloc` returning NULL), `DB_ERR_IO` on a failed `fprintf`, and
the `db_strerror` cases nothing calls. Those are precisely the paths that
run only when something has already gone wrong — the ones that must work and
never get exercised.

Reporting that number rather than hiding it is the point. A capstone that
claims 100% coverage is either trivial or lying.

## CI

```yaml
# .github/workflows/ci.yml
jobs:
  build-and-test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest]
        cc: [gcc, clang]
    runs-on: ${{ matrix.os }}
    env:
      CC: ${{ matrix.cc }}
    steps:
      - uses: actions/checkout@v4
      - name: Build (warnings are errors)
        run: make
      - name: Unit tests
        run: make check
      - name: Tests under ASan + UBSan
        run: make asan
```

Four combinations — two compilers, two operating systems — because gcc and
clang disagree about which warnings to emit, and Linux and macOS disagree
about `char` signedness, struct padding and libc behaviour. `fail-fast:
false` runs all four so you see every failure at once. A second job runs the
coverage report.

That `env: CC` line works only because the Makefile declares `CC ?= cc`
rather than `CC = cc` — the `?=` from
[module 09](09-build-systems-at-scale.md), doing real work.

## Stretch goals

1. **Real durability.** `fflush` hands bytes to the kernel; a power cut still
   loses them. Add `fsync(fileno(db->wal))` behind a `db_set_sync(Db *, int)`
   flag, then measure inserts per second with it on and off. The ratio (often
   100x or worse) is why every database exposes this as a tunable rather than
   choosing for you.

2. **Compaction.** Tombstones and an append-only log both grow without bound.
   Write `db_compact()` that rewrites the WAL containing only live rows, and
   frees tombstoned entries — which forces you to confront the borrowed
   `const Row *` contract and document exactly when a pointer from `db_get`
   becomes invalid.

3. **Crash safety, tested properly.** Truncate the WAL at a random byte
   offset and confirm recovery discards the partial trailing record instead
   of accepting a corrupt row. Then add a CRC to each record and a
   `DB_ERR_CORRUPT` status. Script it: 1,000 random truncations, zero
   crashes, zero silently-wrong recoveries.

4. **Concurrency.** Put the engine behind the thread-per-connection socket
   server from [Level 3 module 10](../level-3/10-project-kv-store.md) with a
   `pthread_rwlock`. Verify with `-fsanitize=thread` under 20 parallel
   `curl`s, then move the WAL write outside the lock and explain — with a
   test — why record ordering becomes a problem.

5. **A real query language.** Extend the parser to `SELECT name, score WHERE
   score > 50 ORDER BY score DESC LIMIT 10`. Build an AST, then an
   interpreter over it. Fuzz the parser with libFuzzer
   ([module 08](08-testing-in-c.md)) until 100,000 random inputs produce zero
   crashes and zero sanitizer reports.

6. **Secondary indexes.** Add a B-tree or skip list keyed on `score` so range
   queries stop being O(n) scans. Benchmark `SELECT ... WHERE score > x`
   against the linear version at 1,000 / 100,000 / 1,000,000 rows, and find
   the row count where the index starts winning — it is larger than most
   people guess.

7. **Ship it as a library.** Build `libtinydb.so`, wrap it with Python
   `ctypes` ([module 07](07-interfacing-other-languages.md)) exposing the
   opaque handle as a class with a context manager, and publish the header
   with proper `extern "C"` guards so C++ callers work too.

---

You have reached the end of the path. What you have built here — an engine
with a defined API, an error model, durability, tests that run clean under
sanitizers, two build systems and CI on four platform combinations — is the
shape of production C. The language gives you nothing for free, so
everything in that list is a decision someone has to make deliberately.
Making them on purpose, and being able to defend each one, is what
distinguishes C that ships from C that merely compiles.
