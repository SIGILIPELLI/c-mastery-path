---
title: "C Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free C cheat sheet: 19 quick-reference tables from every level of the C Mastery Path, each linked to its full lesson."
---

# C Cheat Sheet

Quick-reference tables from **19 lessons** of the free C Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [05 · Arrays &amp; Strings](level-1/05-arrays-strings.md)

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

## Level 2 · Intermediate

### [02 · Dynamic Memory](level-2/02-dynamic-memory.md)

| Call | Returns | Initialized? | Notes |
|------|---------|--------------|-------|
| `malloc(n)` | `n` bytes, or `NULL` | no — garbage | fastest; you must fill it |
| `calloc(c, n)` | `c * n` bytes, or `NULL` | yes — all zero bytes | checks `c * n` for overflow |
| `realloc(p, n)` | `n` bytes, or `NULL` | old contents preserved | may move the block; use a temp |
| `free(p)` | nothing | — | `free(NULL)` is safe and does nothing |

## Level 3 · Advanced

### [02 · Trees &amp; Basic Graph Representations](level-3/02-trees-graphs.md)

| Term | Meaning |
|------|---------|
| Root | The tree's single entry node with no parent |
| Leaf | A node with no children |
| Depth of a node | Number of edges from the root to that node |
| Height of a tree | Depth of its deepest leaf |
| Balanced | Left and right subtree heights differ by at most a small constant at every node |
| Directed vs. undirected graph | Whether an edge `A → B` implies `B → A` (undirected, as used above) or not |
| Adjacency matrix | O(V²) space, O(1) "is there an edge?" check |
| Adjacency list | O(V + E) space, O(degree) "is there an edge?" check |
| BFS | Explores level by level; finds shortest path in an unweighted graph |
| DFS | Explores as deep as possible before backtracking; simpler with recursion |

### [03 · Sorting &amp; Searching Algorithms](level-3/03-sorting-searching.md)

| Algorithm | Best | Average | Worst | Notes |
|-----------|------|---------|-------|-------|
| Quicksort | O(n log n) | O(n log n) | O(n²) | In-place; worst case on already-sorted input with a naive pivot |
| Merge sort | O(n log n) | O(n log n) | O(n log n) | Stable, but needs O(n) extra space |
| Bubble sort | O(n) | O(n²) | O(n²) | Simple, never used in practice above tiny n |
| Linear search | O(1) | O(n) | O(n) | Works on unsorted data |
| Binary search | O(1) | O(log n) | O(log n) | Requires sorted data |

### [04 · Bit Manipulation](level-3/04-bit-manipulation.md)

| Task | Expression |
|---|---|
| Set bit n | `x \|= (1u << n)` |
| Clear bit n | `x &= ~(1u << n)` |
| Toggle bit n | `x ^= (1u << n)` |
| Check bit n | `(x & (1u << n)) != 0` |
| Clear lowest set bit | `x &= (x - 1)` |
| Isolate lowest set bit | `x & (-x)` (two's complement) |
| Check power of two | `x != 0 && (x & (x - 1)) == 0` |
| Count set bits | Kernighan's loop, or `__builtin_popcount(x)` on gcc/clang |

### [05 · Memory Management Deep Dive](level-3/05-memory-management-deep-dive.md)

| Bug | Symptom | Tool that catches it |
|---|---|---|
| Use-after-free | Sometimes silent, sometimes a crash later | ASan (`-fsanitize=address`), valgrind |
| Double-free | Allocator corruption, may crash immediately or later | ASan, valgrind |
| Buffer overflow (heap) | Silent corruption of adjacent data | ASan, valgrind |
| Buffer overflow (stack) | Crash, or a classic security vulnerability | ASan, `-fstack-protector` |
| Memory leak | No crash, just growing memory use | valgrind (Linux), `leaks`/Instruments (macOS) |
| Uninitialized read | Nondeterministic behavior | `-fsanitize=undefined`, valgrind's memcheck |

| Function | What it does | Failure return |
|---|---|---|
| `malloc(n)` | Allocates `n` uninitialized bytes | `NULL` |
| `calloc(n, size)` | Allocates `n * size` bytes, zeroed | `NULL` |
| `realloc(p, n)` | Resizes block `p` to `n` bytes, may move it | `NULL` (original block untouched) |
| `free(p)` | Releases the block; `free(NULL)` is a defined no-op | — |

### [06 · Static &amp; Dynamic Libraries](level-3/06-static-dynamic-libraries.md)

| | Static (`.a`) | Dynamic (`.so`/`.dylib`) |
|---|---|---|
| When code is linked | At compile/link time | At process startup (or via `dlopen`, on demand) |
| Copies of the code on disk | One per binary that links it | One, shared by every process |
| Executable size | Larger (includes the library) | Smaller (library is separate) |
| Runtime dependency | None — self-contained | Library file must be present and findable |
| Patch a bug in the library | Recompile and relink every dependent binary | Replace the one shared file |
| Build flags | `ar rcs`, then link with `-L -l` | `-fPIC` to compile, `-shared` to link the library |
| Inspect dependencies | `nm libname.a` | `ldd` (Linux) / `otool -L` (macOS) |

### [07 · Concurrency Basics](level-3/07-concurrency-basics.md)

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

### [08 · Networking Basics](level-3/08-networking-basics.md)

| Call | Purpose |
|---|---|
| `socket(AF_INET, SOCK_STREAM, 0)` | New TCP/IPv4 endpoint (`SOCK_DGRAM` for UDP) |
| `bind(fd, addr, len)` | Claim a local port; needs `SO_REUSEADDR` to restart cleanly |
| `listen(fd, backlog)` | Begin queueing connections |
| `accept(fd, &peer, &len)` | Return a **new** fd for one client |
| `connect(fd, addr, len)` | Client-side: reach out to a server |
| `recv(fd, buf, n, 0)` | Read; `0` means peer closed, `-1` means error |
| `send(fd, buf, n, 0)` | Write; may send **fewer** bytes than asked — loop |
| `close(fd)` | Ends the connection (sends FIN) |
| `htons` / `htonl` | Host → network byte order (16-bit / 32-bit) |
| `ntohs` / `ntohl` | Network → host byte order |
| `inet_pton` / `inet_ntop` | Text address ↔ binary; the `_ntoa`/`_addr` versions are legacy |
| `setsockopt(..., SO_REUSEADDR, ...)` | Rebind a port still in `TIME_WAIT` |

Traps worth re-reading before you debug for an hour:

- `recv` does **not** null-terminate. Always `buf[n] = '\0'` with room
  reserved (`sizeof buf - 1`).
- `send` returning less than `n` is normal on a busy socket. The loop in
  `echo_server.c` is the minimum correct form.
- Writing to a socket whose peer has closed raises `SIGPIPE`, killing your
  process by default. Servers set `signal(SIGPIPE, SIG_IGN)` and check
  `errno == EPIPE` instead.
- `struct sockaddr_in` must be zeroed before use; `= {0}` does it. Leftover
  stack garbage in the padding has caused real bind failures.

### [09 · Working with Unix System Calls](level-3/09-unix-system-calls.md)

| Call | Purpose | Failure |
|---|---|---|
| `open(path, flags, mode)` | Get an fd; `O_CREAT` needs `mode` | `-1` |
| `read(fd, buf, n)` / `write(fd, buf, n)` | Raw I/O; may be short | `-1`; `read` returns `0` at EOF |
| `lseek(fd, off, whence)` | Move the offset; `SEEK_SET/CUR/END` | `(off_t)-1` |
| `close(fd)` | Release the fd — check it, writes can fail here | `-1` |
| `stat` / `fstat` / `lstat` | Metadata by path / fd / link itself | `-1` |
| `fork()` | Duplicate the process; returns twice | `-1` |
| `execlp` / `execvp` | Replace the program image; never returns on success | returns only on failure |
| `waitpid(pid, &st, 0)` | Block for a child and reap it | `-1` |
| `WIFEXITED` / `WEXITSTATUS` | Decode a packed wait status | — |
| `pipe(fds)` | `fds[0]` read end, `fds[1]` write end | `-1` |
| `dup2(old, new)` | Make `new` refer to `old`'s file | `-1` |
| `perror(msg)` / `strerror(errno)` | Turn `errno` into text | — |
| `_exit(n)` | Exit without flushing stdio — for failed `exec` children | — |

## Level 4 · Master

### [01 · Advanced Memory Management](level-4/01-advanced-memory-management.md)

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

### [02 · Concurrency &amp; Synchronization](level-4/02-concurrency-synchronization.md)

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

### [03 · Performance Optimization &amp; Profiling](level-4/03-performance-profiling.md)

| Tool | Kind | What it gives you |
|---|---|---|
| `clock_gettime(CLOCK_MONOTONIC)` | Manual | Trustworthy wall time for a region |
| `-fprofile-instr-generate -fcoverage-mapping` + `llvm-cov show` | Instrumented | Exact per-line execution counts |
| `gprof` (with `-pg`) | Instrumented | Call counts and flat profile; Linux/GCC |
| `perf record` / `perf report` | Sampling | Where time goes in a release build (Linux) |
| `sample <pid>` | Sampling | Same, on macOS; also `xctrace` for Instruments |
| `perf stat` | Counters | Cache misses, branch mispredicts, IPC |
| `-O2` | Compiler | The sane default; `-O3` rarely adds much |
| `-march=native` | Compiler | Uses this CPU's instructions — breaks portability |
| `-flto` | Compiler | Cross-file inlining at link time |
| `-fno-omit-frame-pointer` | Compiler | Keeps profiler stacks readable |

Traps that specifically bite C programmers here:

- **Timing a `-O0` build.** Debug builds can be 5–10x slower and have
  completely different hotspots. Profile what you ship.
- **Benchmarking code the optimiser deleted.** If a result is unused, the
  loop computing it can vanish entirely — you measure nothing and conclude
  it is infinitely fast. Always consume the result (print it, accumulate it
  into a `volatile`), as every example here does.
- **Measuring once.** Run each variant several times. The 83x above held
  across runs; a 5% difference across single runs is noise.
- **Optimising before profiling.** The `strcmp` in `find_slot` was never the
  problem. Replacing it with a faster comparison would have won a few
  percent of a fundamentally wrong design.

### [04 · Writing Portable C](level-4/04-writing-portable-c.md)

| Need | Portable answer |
|---|---|
| Exact-width integer | `int32_t`, `uint64_t` (`<stdint.h>`) |
| Size of an object / an index | `size_t` |
| Difference of two pointers | `ptrdiff_t` |
| Integer holding a pointer | `intptr_t` / `uintptr_t` |
| Print them | `%zu`, `%td`, `PRId64`, `PRIxPTR` (`<inttypes.h>`) |
| Byte data | `unsigned char`, never plain `char` |
| Type limits | `<limits.h>`, `<float.h>` |
| Alignment of a type | `_Alignof(T)` (C11) |
| Offset of a member | `offsetof(T, m)` (`<stddef.h>`) |
| Serialise a number | Shift and mask into `unsigned char[]` |
| Detect C version | `__STDC_VERSION__` (`201112L` = C11, `201710L` = C17) |
| Detect platform | `_WIN32`, `__linux__`, `__APPLE__`, `__unix__` |
| Enable POSIX APIs | `#define _POSIX_C_SOURCE 200809L` first |

Portability flags worth turning on: `-std=c11 -pedantic -Wall -Wextra`
rejects extensions you did not mean to use, and
`-Wconversion -Wsign-conversion` finds the implicit narrowing that silently
changes behaviour when a type's width changes underneath you.

### [05 · Security in C](level-4/05-security-in-c.md)

| Never use | Use instead | Why |
|---|---|---|
| `gets()` | `fgets()` | No bound at all; removed from C11 |
| `strcpy` / `strcat` | `snprintf` | No bound |
| `sprintf` | `snprintf` | No bound |
| `strncpy` | `snprintf` | May not null-terminate |
| `printf(user)` | `printf("%s", user)` | Format string attack |
| `atoi` | `strtol` + `errno` | No error or overflow reporting |
| `malloc(n * sz)` | `calloc(n, sz)` or check first | Multiplication overflow |
| `scanf("%s", buf)` | `scanf("%15s", buf)` / `fgets` | Unbounded write |

| Flag | Effect |
|---|---|
| `-fsanitize=address` | Catches overflow, use-after-free, double free |
| `-fsanitize=undefined` | Catches signed overflow, bad shifts, misaligned access |
| `-D_FORTIFY_SOURCE=2 -O2` | Compile-time-known bounds checked at runtime |
| `-fstack-protector-strong` | Detects stack smashing via a canary |
| `-Wformat-security -Werror=format-security` | Rejects non-literal format strings |
| `-Wall -Wextra -Wconversion` | Catches the narrowing that precedes overflow |
| `-fPIE -pie`, `-Wl,-z,relro,-z,now` | ASLR and hardened relocations (Linux) |

The other three bug classes to internalise:

- **Use-after-free / double free.** A pointer is not invalidated by `free`.
  Set it to `NULL` immediately after freeing; `free(NULL)` is a guaranteed
  no-op, so a double free becomes harmless.
- **Off-by-one.** `for (i = 0; i <= n; i++)` on an `n`-element array writes
  one past the end — often exactly the byte holding a length, a flag, or a
  saved pointer, as `escalate.c` showed.
- **TOCTOU.** Checking a file with `access()` and then `open()`ing it lets an
  attacker swap the file in between. Open first, then check the *file
  descriptor* with `fstat`.

Finally: **all input is hostile**, including input from files you wrote,
environment variables, `argv`, and other processes on the same machine.
Validate length before copying, range before indexing, and sign before
converting to `size_t` — a negative `int` becomes an enormous unsigned value
and turns a length check into a no-op.

### [06 · Advanced Data Structures](level-4/06-advanced-data-structures.md)

| Structure | Lookup | Insert | Space | Use when |
|---|---|---|---|---|
| Sorted array | O(log n) | O(n) | Minimal | Built once, read often |
| Plain BST | O(n) worst | O(n) worst | 2 ptrs/node | Never, if input may be ordered |
| AVL tree | O(log n) | O(log n) | 2 ptrs + int | Lookup-heavy, ordered iteration needed |
| Red-black tree | O(log n) | O(log n) | 2 ptrs + bit | Insert/delete-heavy ordered map |
| Hash table (chained) | O(1) avg | O(1) avg | Buckets + nodes | No ordering needed |
| Hash table (open addr.) | O(1) avg | O(1) avg | One array | Cache-friendly, load factor < 0.7 |
| Trie | O(len) | O(len) | High | Prefix queries, autocomplete |
| Skip list | O(log n) avg | O(log n) avg | ~2 ptrs/node | Simpler than a tree, lock-free friendly |
| Bloom filter | O(k) | O(k) | ~10 bits/item | "Definitely not present?" |

Choosing between them, honestly: a hash table beats a balanced tree for
plain key-value lookup nearly every time, and the tree earns its keep only
when you need **ordering** — range queries, in-order iteration, nearest-key.
If you find yourself sorting a hash table's contents on every read, you
wanted a tree.

Two C-specific implementation notes that apply across all of them:

- Prefer **intrusive** structures for performance: put the `next`/`left`/
  `right` pointers inside the element, recover the containing struct with
  `offsetof`, and you eliminate one allocation and one pointer chase per
  node. That is what the Linux kernel's `list_head` does.
- Recursive tree code is clear but stack-bounded. It is safe for balanced
  structures (depth ~20 at a million nodes) and dangerous for anything that
  might degenerate. Convert to an explicit stack when the depth is not
  provably logarithmic.

### [07 · Interfacing with Other Languages](level-4/07-interfacing-other-languages.md)

| Concern | Rule |
|---|---|
| Symbol names | C only. C++ needs `extern "C"` to avoid mangling |
| Build | `-fPIC -shared`; `.so` (Linux), `.dylib` (macOS), `.dll` (Windows) |
| Types at the boundary | Only `<stdint.h>` fixed-width types, `double`, and pointers |
| `int` / `long` | Avoid — width varies by platform ([module 04](level-4/04-writing-portable-c.md)) |
| Structs | Layout must match exactly; padding is implementation-defined |
| Strings | C: null-terminated bytes. Encode/decode explicitly, never assume UTF-8 |
| Errors | Return codes or an out-parameter. Never exceptions, never `longjmp` |
| Ownership | Whoever allocates, frees. Export a `free_*` for anything you return |
| Callbacks | Keep the host-side object alive; never let exceptions escape |
| Threads | Document whether your functions are thread-safe; the host may call from many |

| Binding tool | Notes |
|---|---|
| Python `ctypes` | Stdlib, no build step, runtime-only checking — used here |
| Python `cffi` | Parses real C declarations; safer, needs installing |
| Python C API | Fastest, most work, ties you to CPython versions |
| Rust | `extern "C"` + `#[repr(C)]`; `bindgen` generates declarations from headers |
| Go | `cgo`; note the real cost per call and the pointer-passing rules |
| Java | JNI (verbose) or the newer FFM API (`java.lang.foreign`) |
| Node.js | N-API |

### [08 · Testing in C](level-4/08-testing-in-c.md)

| Technique | Catches |
|---|---|
| Unit tests on boundaries (0, 1, n, n+1, NULL) | Off-by-one, empty-input handling |
| `-fsanitize=address` | Overflow, use-after-free, double free, leaks |
| `-fsanitize=undefined` | Signed overflow, bad shift, misaligned access |
| `-fsanitize=thread` | Data races that did not manifest |
| `llvm-cov` / `gcov` branch coverage | Code you never exercised |
| Property tests | Cases you never imagined |
| libFuzzer / AFL++ | Deep paths reachable only by odd input |
| `assert()` in the code itself | Broken internal invariants, in debug builds |

| Framework | Notes |
|---|---|
| Hand-rolled macros | Zero dependencies; what this module built |
| Unity | Tiny, embedded-friendly, C89 |
| Check | Forks per test, so a crash does not kill the run |
| Criterion | Auto-registers tests, parameterised, modern |
| CMocka | Mocking and fixtures |

Testing traps specific to C:

- **`assert()` vanishes under `-DNDEBUG`.** Never put an operation with side
  effects inside one — `assert(pop(&stack) == 3)` stops popping in release
  builds.
- **A test that crashes takes the runner with it.** Frameworks like Check
  fork per test for exactly this reason; hand-rolled runners lose every
  result after the crash.
- **Global state persists between tests.** A file-scope table left dirty by
  test 3 makes test 7 pass for the wrong reason. Reset explicitly in setup,
  and run tests in a different order sometimes to check.
- **Testing only the `-O0` build.** Undefined behaviour frequently changes
  shape with optimisation, as [module 02](level-4/02-concurrency-synchronization.md)
  showed. Run the suite at `-O0` and `-O2`.

### [09 · Build Systems at Scale](level-4/09-build-systems-at-scale.md)

| Need | Make | CMake |
|---|---|---|
| Header deps | `-MMD -MP` + `-include $(DEP)` | Automatic |
| Out-of-source | `$(BUILD)/%.o: %.c` | `-S . -B dir` |
| Parallel build | `make -j$(nproc)` | `cmake --build dir -j8` |
| Run tests | Custom `check:` target | `enable_testing()` + `ctest` |
| Variant build | Target-specific variables | `-DOPTION=ON`, separate build dir |
| Debug the build | `make -n` (dry run), `make -p` | `cmake --build . -v` |

| Flag / tool | Purpose |
|---|---|
| `-MMD -MP` | Generate header dependencies (`-MD` also tracks system headers) |
| `make -j8` | Parallel compile; the biggest single speedup available |
| `ccache` | Cache object files across clean builds |
| `ninja` (`cmake -G Ninja`) | Much faster than Make for large graphs |
| `-flto` | Cross-translation-unit inlining at link time |
| `compile_commands.json` | Feeds `clangd`, `clang-tidy`, IDEs |

Build-system traps, in the order they cost people time:

- **Mixing objects built with different flags.** An ASan object linked with
  a non-ASan one produces link errors at best and corruption at worst. Every
  flag change needs a separate directory or a `clean`.
- **A recursive Make per directory.** Each sub-make sees only part of the
  dependency graph, so it cannot parallelise correctly or detect
  cross-directory staleness. Use one Make with `include`d fragments.
- **Ignoring `-j` correctness.** A rule whose recipe writes a file some other
  rule reads, without a declared dependency, works serially and fails
  randomly at `-j8`. If `make -j` fails but `make` works, you have a missing
  dependency, not a flaky compiler.
- **Rules with tab/space confusion.** Make requires a literal tab to start a
  recipe line. This is still, decades later, the most common first Makefile
  error.

---

Go deeper with the full [C Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
