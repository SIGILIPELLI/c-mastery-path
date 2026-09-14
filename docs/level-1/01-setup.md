# 01 · Setup & First Program

## 🎥 Video walkthrough

<iframe width="100%" height="400" style="max-width:720px;aspect-ratio:16/9;height:auto;" src="https://www.youtube.com/embed/X2qYY2dN4h8" title="Video walkthrough" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

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

## How It Actually Works

`gcc hello.c -o hello` is not one step, it's a pipeline of four separate
programs chained together, each of which you can run by hand:

1. **Preprocessing** (`cpp`) — textually expands `#include`, macros, and
   conditionals. `gcc -E hello.c` dumps the result: you'd see the entire
   contents of `stdio.h` (hundreds of lines of function prototypes and type
   declarations) pasted in above your `main`, with the `#include` line gone.
2. **Compiling to assembly** (`cc1`) — translates the preprocessed C into
   architecture-specific assembly text. Run `gcc -S hello.c` and open the
   resulting `hello.s`; you'll see instructions like `call printf` and a
   `leaq` loading the address of your string literal into a register before
   the call, plus a `main:` label and a `ret`.
3. **Assembling** (`as`) — turns that assembly text into machine code bytes,
   producing an object file (`gcc -c hello.c` → `hello.o`). This file has
   raw instruction bytes but is not yet runnable: it has unresolved
   references to things like `printf`, which live in a separate library.
4. **Linking** (`ld`) — resolves `printf` by pulling in the C standard
   library (`libc`), and merges everything into one executable with a
   correct entry point. On macOS/Linux the OS loader expects a specific
   binary format (Mach-O or ELF) with headers describing where code, string
   constants, and other segments live in the final file.

Only the last stage's output — the ELF/Mach-O binary — is what
`./hello` actually executes. When you run it, the OS's loader reads those
headers, maps the code segment into a fresh process's address space as
read-only+executable memory, maps a separate writable segment for globals,
sets up a stack, and jumps the CPU's instruction pointer to `main`'s
address. The string `"Hello, world!\n"` isn't "in a variable" the way Java
would box it — it's a sequence of bytes baked directly into the binary's
read-only data segment at compile time, and `printf` is handed a raw pointer
to the first byte of that sequence.

`return 0;` doesn't just end the function — it sets the CPU's return-value
register (`%eax` on x86, `w0` on ARM64) to `0`, and the shell reads that
register's value as the process's exit status via the `wait()`/`waitpid()`
system call family, which is exactly what lets `&&` in
`gcc hello.c -o hello && ./hello` decide whether to run the second command.

## Exercise

Write a program `greet.c` that uses three separate `printf` calls to print a
greeting, your name, and a farewell message, each on its own line. Compile it
with `gcc greet.c -o greet` and run the resulting binary. Then try renaming the
output binary (`-o mygreeting`) and confirm the program still runs the same
way under the new name.
