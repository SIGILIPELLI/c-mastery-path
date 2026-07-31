# 08 · Makefiles & Build Systems

Once a project has more than three source files, typing `gcc a.c b.c c.c -o app`
gets old — and it recompiles everything even when you changed one line.
`make` fixes both problems. You describe *what depends on what*, and `make`
figures out the minimum set of commands needed to bring the build up to date,
by comparing file timestamps.

`make` is decades old, present on every Unix system, and still the backbone of
the Linux kernel, SQLite, and countless C projects. Even when a project uses
CMake or Meson, those generate Makefiles underneath.

## The anatomy of a rule

```make
target: prerequisites
	recipe
```

- **target** — the file to produce (or a name like `clean`).
- **prerequisites** — files it depends on. If any is *newer* than the target,
  the target is stale.
- **recipe** — shell commands to rebuild it. **Must be indented with a real TAB,
  not spaces.** This is the single most common Makefile error, and the message
  (`missing separator`) does not mention tabs.

## A first Makefile

Assume the module layout from [Module 7](07-modular-programming.md):
`main.c`, `counter.c`, `counter.h`.

```make
# Makefile
app: main.o counter.o
	gcc main.o counter.o -o app

main.o: main.c counter.h
	gcc -Wall -Wextra -c main.c -o main.o

counter.o: counter.c counter.h
	gcc -Wall -Wextra -c counter.c -o counter.o

clean:
	rm -f app main.o counter.o
```

```bash
$ make
gcc -Wall -Wextra -c main.c -o main.o
gcc -Wall -Wextra -c counter.c -o counter.o
gcc main.o counter.o -o app

$ make                    # nothing changed
make: 'app' is up to date.

$ touch counter.c && make # only the affected parts rebuild
gcc -Wall -Wextra -c counter.c -o counter.o
gcc main.o counter.o -o app
```

Running `make` with no arguments builds the **first** target in the file, so put
your main artifact at the top. Note that touching `counter.h` would rebuild
*both* object files — that's the dependency graph doing its job.

## Variables kill the repetition

```make
CC      := gcc
CFLAGS  := -Wall -Wextra -std=c11 -g
LDFLAGS :=
TARGET  := app
OBJS    := main.o counter.o stats.o

$(TARGET): $(OBJS)
	$(CC) $(OBJS) $(LDFLAGS) -o $(TARGET)

clean:
	rm -f $(TARGET) $(OBJS)
```

| Assignment | Behaviour |
|------------|-----------|
| `:=` | Evaluated **once**, immediately. Use this by default. |
| `=` | Re-evaluated every time it's used (recursive; can surprise you) |
| `?=` | Set only if not already defined — lets `make CC=clang` override it |
| `+=` | Append |

`make CFLAGS="-O2" ` on the command line overrides the file's value entirely,
which is how you switch to a release build without editing anything.

## Pattern rules and automatic variables

Writing one rule per `.o` doesn't scale. A **pattern rule** covers them all:

```make
CC      := gcc
CFLAGS  := -Wall -Wextra -std=c11 -g
TARGET  := app
SRCS    := $(wildcard *.c)          # every .c in this directory
OBJS    := $(SRCS:.c=.o)            # main.c counter.c -> main.o counter.o

.PHONY: all clean run

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

run: $(TARGET)
	./$(TARGET)

clean:
	rm -f $(TARGET) $(OBJS)
```

| Automatic variable | Expands to |
|--------------------|------------|
| `$@` | The target being built (`app`, or `main.o`) |
| `$<` | The **first** prerequisite (`main.c`) |
| `$^` | **All** prerequisites, deduplicated (`main.o counter.o`) |
| `$?` | Only the prerequisites newer than the target |
| `$*` | The stem matched by `%` (`main` for `main.o`) |

### `.PHONY` — targets that aren't files

`clean` doesn't produce a file called `clean`. If someone ever creates a file
with that name, `make clean` would report "up to date" and do nothing. Declaring
`.PHONY: clean` tells `make` to always run the recipe. Every non-file target
(`all`, `clean`, `run`, `test`, `install`) should be listed.

## Automatic header dependencies

The pattern rule above has a real bug: `%.o: %.c` never mentions headers, so
editing `counter.h` won't trigger a rebuild — and you get a mysteriously stale
binary. Generating the dependencies with the compiler fixes it permanently:

```make
CC      := gcc
CFLAGS  := -Wall -Wextra -std=c11 -g
DEPFLAGS := -MMD -MP                 # emit a .d file listing every header used

SRCS := $(wildcard *.c)
OBJS := $(SRCS:.c=.o)
DEPS := $(SRCS:.c=.d)

app: $(OBJS)
	$(CC) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) $(DEPFLAGS) -c $< -o $@

-include $(DEPS)                     # leading '-' = ignore if not there yet

.PHONY: clean
clean:
	rm -f app $(OBJS) $(DEPS)
```

`gcc -MMD` writes `main.d` containing something like:

```make
main.o: main.c counter.h stats.h
```

`-include` pulls those generated rules in, so `make` now knows exactly which
headers each object depends on. `-MP` adds harmless empty targets for each
header, so deleting a header doesn't break the build with "no rule to make
target". These four extra characters eliminate an entire category of
"it works after `make clean`" bugs.

## Useful flags to have in `CFLAGS`

| Flag | Why |
|------|-----|
| `-Wall -Wextra` | Turn on the warnings that catch real bugs. Non-negotiable. |
| `-std=c11` | Pin the language standard so builds are reproducible |
| `-g` | Debug symbols, so `gdb` can show source ([Module 9](09-debugging-tools.md)) |
| `-O0` / `-O2` | No optimization while debugging / optimized for release |
| `-Werror` | Treat warnings as errors — good in CI, harsh while learning |
| `-fsanitize=address,undefined` | Catch memory and UB bugs at run time |
| `-pedantic` | Warn about non-standard extensions |

A debug/release split, driven by one variable:

```make
CFLAGS := -Wall -Wextra -std=c11

ifeq ($(BUILD),release)
    CFLAGS += -O2 -DNDEBUG
else
    CFLAGS += -O0 -g -fsanitize=address,undefined
    LDFLAGS += -fsanitize=address,undefined
endif
```

```bash
make                  # debug build with sanitizers
make BUILD=release    # optimized, asserts compiled out
```

Note `-DNDEBUG` in the release branch: that's what disables `assert` from
[Module 6](06-error-handling.md).

## A complete project Makefile

With sources in `src/`, headers in `include/`, and build output in `build/`:

```make
CC       := gcc
CFLAGS   := -Wall -Wextra -std=c11 -g -Iinclude
DEPFLAGS := -MMD -MP

SRC_DIR   := src
BUILD_DIR := build
TARGET    := $(BUILD_DIR)/inventory

SRCS := $(wildcard $(SRC_DIR)/*.c)
OBJS := $(patsubst $(SRC_DIR)/%.c,$(BUILD_DIR)/%.o,$(SRCS))
DEPS := $(OBJS:.o=.d)

.PHONY: all clean run

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $^ -o $@

$(BUILD_DIR)/%.o: $(SRC_DIR)/%.c | $(BUILD_DIR)
	$(CC) $(CFLAGS) $(DEPFLAGS) -c $< -o $@

$(BUILD_DIR):
	mkdir -p $(BUILD_DIR)

run: $(TARGET)
	./$(TARGET)

clean:
	rm -rf $(BUILD_DIR)

-include $(DEPS)
```

The `| $(BUILD_DIR)` is an **order-only prerequisite**: the directory must exist
before compiling, but its timestamp (which changes whenever a file is added)
must not make objects look stale. Without the `|`, every new file would trigger
a full rebuild.

## Useful `make` invocations

| Command | Effect |
|---------|--------|
| `make` | Build the first target |
| `make clean all` | Full rebuild |
| `make -j8` | Run up to 8 recipes in parallel — often a 4–8× speedup |
| `make -n` | Dry run: print the commands without executing them |
| `make -B` | Force rebuild, ignoring timestamps |
| `make --debug=b` | Explain why each target was considered out of date |

## Beyond `make`

Makefiles are hand-written and platform-specific. Larger projects usually add a
generator on top:

| Tool | What it is |
|------|-----------|
| **CMake** | Describes the build abstractly; generates Makefiles, Ninja files, or IDE projects. The de-facto standard for cross-platform C/C++. |
| **Ninja** | A very fast, deliberately dumb build executor. Almost always generated, not written by hand. |
| **Meson** | Modern, readable syntax; generates Ninja files. Popular in the GNOME/GStreamer world. |
| **Autotools** | The classic `./configure && make && make install`. Powerful, portable, and painful. |

Learn `make` first regardless — every one of these ultimately runs the same
compile-and-link commands, and being able to read a Makefile tells you what any
build is actually doing. Scaling builds across many modules and platforms is
picked up again in [Level 4](../level-4/09-build-systems-at-scale.md).

## Exercise

Take the `Stack` module from [Module 7](07-modular-programming.md)'s exercise
and give it a real build. Write a Makefile with `CC`/`CFLAGS` variables, a
pattern rule for `%.o: %.c`, `-MMD -MP` dependency generation, and `.PHONY`
targets `all`, `clean`, `run`, and `test`. Verify three things: (1) running
`make` twice only builds once, (2) `touch stack.h` rebuilds both objects, and
(3) `make -n clean` prints the `rm` command without deleting anything. Then add a
`BUILD=release` branch that swaps `-O0 -g` for `-O2 -DNDEBUG`, and confirm with
`make -n BUILD=release` that the flags actually change.
