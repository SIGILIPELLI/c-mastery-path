# 09 · Build Systems at Scale

A build system for three files is a convenience. A build system for three
hundred is the thing that decides whether your project is pleasant or
miserable to work on, because it answers one question thousands of times a
day: **given what changed, what is the minimum I must rebuild — and is that
answer correct?**

Get it wrong in the cheap direction and you rebuild everything on every
keystroke. Get it wrong in the expensive direction — the far worse failure —
and you link a stale object file against a changed header, producing a
binary that matches no version of your source and crashes in ways that make
no sense.

## The project

```text
include/mathx.h   include/strx.h
src/mathx.c       src/strx.c      src/main.c
tests/test_mathx.c
```

Small, but with the structure that matters: a library, an application, tests
that link the library but not `main.o`, and headers shared between them.

## Make, done properly

The single most important feature is **automatic header dependency
tracking**. Hand-written rules like `main.o: main.c mathx.h` are always
wrong eventually, because someone adds an `#include` and forgets. Let the
compiler generate them:

```makefile
CC       ?= cc
CFLAGS   ?= -std=c11 -Wall -Wextra -O2
CPPFLAGS += -Iinclude -MMD -MP          # -MMD/-MP generate the .d files
LDFLAGS  ?=

BUILD    := build
SRC      := $(wildcard src/*.c)
OBJ      := $(SRC:%.c=$(BUILD)/%.o)
DEP      := $(OBJ:.o=.d)

LIB_OBJ  := $(filter-out $(BUILD)/src/main.o,$(OBJ))
TEST_SRC := $(wildcard tests/*.c)
TEST_BIN := $(TEST_SRC:tests/%.c=$(BUILD)/%)

APP      := $(BUILD)/app

all: $(APP)

$(APP): $(OBJ)
	@mkdir -p $(@D)
	$(CC) $(LDFLAGS) -o $@ $^

$(BUILD)/%.o: %.c
	@mkdir -p $(@D)
	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@

$(BUILD)/%: tests/%.c $(LIB_OBJ)
	@mkdir -p $(@D)
	$(CC) $(CFLAGS) $(CPPFLAGS) -o $@ $< $(LIB_OBJ) $(LDFLAGS)

check: $(TEST_BIN)
	@for t in $(TEST_BIN); do echo "--- $$t"; ./$$t || exit 1; done

asan: CFLAGS  += -fsanitize=address,undefined -g -O1
asan: LDFLAGS += -fsanitize=address,undefined
asan: clean check

clean:
	rm -rf $(BUILD)

-include $(DEP)                          # pull in the generated dependencies

.PHONY: all check clean asan
```

```bash
make
```

```text
cc -std=c11 -Wall -Wextra -O2 -Iinclude -MMD -MP -c src/main.c -o build/src/main.o
cc -std=c11 -Wall -Wextra -O2 -Iinclude -MMD -MP -c src/mathx.c -o build/src/mathx.o
cc -std=c11 -Wall -Wextra -O2 -Iinclude -MMD -MP -c src/strx.c -o build/src/strx.o
cc  -o build/app build/src/main.o build/src/mathx.o build/src/strx.o
```

```bash
make          # again, with nothing changed
```

```text
make: Nothing to be done for `all'.
```

`-MMD` wrote a dependency file next to each object:

```text
$ cat build/src/main.d
build/src/main.o: src/main.c include/mathx.h include/strx.h
include/mathx.h:
include/strx.h:
```

The first line is a real Make rule, pulled in by `-include $(DEP)`. Now
touch a header and watch the *correct* subset rebuild:

```bash
touch include/mathx.h
make
```

```text
cc ... -c src/main.c  -o build/src/main.o
cc ... -c src/mathx.c -o build/src/mathx.o
cc  -o build/app build/src/main.o build/src/mathx.o build/src/strx.o
```

`main.c` and `mathx.c` rebuilt because both include `mathx.h`. **`strx.c` did
not**, because it does not. That is the entire value proposition, and it was
derived by the compiler rather than maintained by hand.

Details in that Makefile worth stealing:

- **`-MP`** emits those bare `include/mathx.h:` targets. Without it, deleting
  a header makes Make fail with "no rule to make target" instead of just
  rebuilding.
- **`-include`** (with the leading dash) does not error on the first build,
  when no `.d` files exist yet.
- **`?=` for `CC`/`CFLAGS`, `+=` for `CPPFLAGS`** lets a caller or CI
  override the compiler (`make CC=clang`) without editing the file.
- **Target-specific variables** (`asan: CFLAGS += ...`) give a whole variant
  build in three lines. `asan` depends on `clean` because objects built with
  different flags must not be mixed.
- **`.PHONY`** stops `make clean` from being confused by a file named
  `clean`.
- **`$(@D)`** is the directory of the target, so `mkdir -p` creates the
  object tree on demand and objects stay out of the source tree.

```bash
make check
```

```text
cc ... -o build/test_mathx tests/test_mathx.c build/src/mathx.o build/src/strx.o
--- build/test_mathx
test_mathx: passed
```

## CMake, for when Make stops scaling

Make struggles with multiple platforms, multiple compilers, finding
dependencies, and generating IDE projects. CMake describes *what* you build
and generates the build files for whatever tool is present.

```cmake
cmake_minimum_required(VERSION 3.16)
project(toolchain_demo VERSION 1.0 LANGUAGES C)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)     # generates compile_commands.json

add_library(core STATIC src/mathx.c src/strx.c)
# PUBLIC: propagates to everything that links `core`
target_include_directories(core PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
target_compile_options(core PRIVATE -Wall -Wextra)

add_executable(app src/main.c)
target_link_libraries(app PRIVATE core)   # include dirs come along automatically

enable_testing()
add_executable(test_mathx tests/test_mathx.c)
target_link_libraries(test_mathx PRIVATE core)
add_test(NAME mathx COMMAND test_mathx)

option(ENABLE_ASAN "Build with AddressSanitizer" OFF)
if(ENABLE_ASAN)
  foreach(t core app test_mathx)
    target_compile_options(${t} PRIVATE -fsanitize=address,undefined -g)
    target_link_options(${t}    PRIVATE -fsanitize=address,undefined)
  endforeach()
endif()
```

```bash
cmake -S . -B cmake-build -DCMAKE_BUILD_TYPE=Release
cmake --build cmake-build
```

```text
[ 14%] Building C object CMakeFiles/core.dir/src/mathx.c.o
[ 28%] Building C object CMakeFiles/core.dir/src/strx.c.o
[ 42%] Linking C static library libcore.a
[ 57%] Building C object CMakeFiles/app.dir/src/main.c.o
[ 71%] Linking C executable app
[ 85%] Building C object CMakeFiles/test_mathx.dir/tests/test_mathx.c.o
[100%] Linking C executable test_mathx
```

```bash
cd cmake-build && ctest --output-on-failure
```

```text
    Start 1: mathx
1/1 Test #1: mathx ............................   Passed    0.48 sec

100% tests passed out of 1
```

The concept that makes modern CMake worth learning is **`PUBLIC` vs
`PRIVATE` usage requirements**. `target_include_directories(core PUBLIC
include)` means "I need this to compile, and so does anyone who links me."
So `app` never mentions `include/` — it links `core` and inherits the path.
`PRIVATE` means the requirement stops at this target. Getting this right is
what makes a large CMake project maintainable: dependencies flow through the
graph instead of being repeated in every target.

`-S . -B cmake-build` is an **out-of-source build**: nothing generated ever
touches your source tree, and `rm -rf cmake-build` is a guaranteed-complete
clean. It also means variants coexist:

```bash
cmake -S . -B cmake-asan -DENABLE_ASAN=ON
cmake --build cmake-asan && (cd cmake-asan && ctest)
```

```text
100% tests passed out of 1
```

Two configurations, two directories, neither invalidating the other's
objects.

`CMAKE_EXPORT_COMPILE_COMMANDS` is worth turning on unconditionally. It
writes `compile_commands.json`:

```json
{
  "directory": ".../cmake-build",
  "command": "/usr/bin/cc -I.../include -O3 -DNDEBUG -std=gnu11 -Wall -Wextra
              -o CMakeFiles/core.dir/src/mathx.c.o -c .../src/mathx.c",
  "file": ".../src/mathx.c"
}
```

That file is what `clangd`, `clang-tidy`, and every serious editor use to
know your include paths and macros. Without it, your IDE is guessing.

## Cheat sheet

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

## Exercise

Add a CI workflow that builds this project three ways and would have caught
every bug in Level 4. Create `.github/workflows/ci.yml` with a matrix over
`{gcc, clang}` × `{ubuntu-latest, macos-latest}` that runs: a normal
`-O2 -Wall -Wextra -Werror` build plus tests, a
`-fsanitize=address,undefined` build plus tests, and a coverage run that
fails if branch coverage drops below a threshold you pick.

Then measure what the build system actually costs you. Time a cold build,
then `make -j8`, then a rebuild with `ccache` warm, then the same via
`cmake -G Ninja`. Record all four numbers. Finally, deliberately break the
dependency tracking — remove `-MMD -MP` and the `-include $(DEP)` line —
then edit a header in a way that changes a struct's layout (add a field at
the front), rebuild, and run the tests. The crash you get is the reason that
line exists, and it is worth seeing once.
