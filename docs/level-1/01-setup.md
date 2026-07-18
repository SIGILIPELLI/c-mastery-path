# 01 · Setup & First Program

## Install a compiler

C source code is just text — a compiler turns it into a binary your machine can
run. Unlike Java, there's no single official toolchain; you'll use `gcc` or
`clang` depending on your platform. Either works fine for this course.

```bash
# macOS -- Xcode Command Line Tools (installs clang, the default on Mac)
xcode-select --install

# macOS -- Homebrew (installs a real gcc, useful once you want GNU extensions)
brew install gcc

# Ubuntu/Debian
sudo apt install build-essential

# Windows -- either MinGW-w64 (native gcc) or, often smoother:
# install WSL (Windows Subsystem for Linux), then run the Ubuntu command above
# inside it.
```

Verify the install:

```bash
gcc --version
# gcc (Homebrew GCC ...) 14.x   -- or "Apple clang" if using Xcode tools

clang --version
# Apple clang version 15.x
```

On macOS, `gcc` is often just an alias for `clang` unless you installed the
real thing via Homebrew — that's fine, both accept the same command-line
options we use in this course.

## Writing the first program

Create a file called `hello.c`:

```c
// hello.c
#include <stdio.h>

int main(void) {
    printf("Hello, world!\n");
    return 0;
}
```

## Compiling and running

```bash
gcc hello.c -o hello
# produces a binary named "hello" (or "hello.exe" on Windows)

./hello
# Hello, world!
```

The `-o hello` flag tells the compiler what to name the output binary. Without
it, gcc defaults to a generic name (`a.out` on macOS/Linux) — naming your
binaries explicitly is worth the two extra keystrokes once you have more than
one program in a directory.

You can also compile and run in one line while experimenting:

```bash
gcc hello.c -o hello && ./hello
```

Real projects skip typing this out by hand every time in favor of a build tool
— we'll get there in [Module 9](09-preprocessor-multifile.md), and Level 2
covers Makefiles properly.

## Anatomy of the program

| Piece | Meaning |
|-------|---------|
| `#include <stdio.h>` | A preprocessor directive that pulls in declarations for standard I/O functions like `printf`, before compilation proper begins. |
| `int main(void)` | The program's entry point. `int` means it returns an integer exit status; `void` means it takes no arguments (there's also an `int main(int argc, char *argv[])` form for command-line arguments, covered later). |
| `printf("Hello, world!\n")` | Prints formatted text to standard output. `\n` is a newline escape sequence, not a literal backslash-n. |
| `return 0;` | Exits `main` with status `0`, the conventional signal to the shell that the program succeeded. A non-zero return signals an error. |
| `;` | Every statement ends with a semicolon — the compiler uses it to know where one statement ends and the next begins. |
| `{ }` | Curly braces delimit blocks — function bodies, loop bodies, if-bodies. |

Notice there's no class wrapping any of this, unlike Java — C has no concept
of objects at the language level. Functions and variables can exist directly
at the top level of a file.

## Choosing an editor

Any plain text editor works, but **VS Code** with the free "C/C++" extension
(from Microsoft) is the most common choice for beginners — it gives you syntax
highlighting, basic IntelliSense, and integrated debugging without much setup.
CLion (paid, free for students) is a heavier IDE some prefer once projects grow
larger. For this course, the terminal plus any editor you're comfortable in is
enough — the compiler is doing the real work, not the editor.

## Exercise

Write a program `greet.c` that uses three separate `printf` calls to print a
greeting, your name, and a farewell message, each on its own line. Compile it
with `gcc greet.c -o greet` and run the resulting binary. Then try renaming the
output binary (`-o mygreeting`) and confirm the program still runs the same
way under the new name.
