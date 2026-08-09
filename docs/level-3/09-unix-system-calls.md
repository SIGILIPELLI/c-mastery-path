# 09 · Working with Unix System Calls

`printf`, `fopen` and `malloc` are library functions: ordinary C code that
lives in libc. Underneath them sit **system calls** — requests that trap
into the kernel, because only the kernel can touch a disk, hand out memory,
or create a process. Knowing where that line falls explains a whole class of
otherwise baffling behaviour: why output appears out of order, why a
buffered line gets printed twice, and why `errno` is meaningless until you
have checked the return value.

The rule for every system call in this module is the same: **it returns -1
on failure and sets `errno`**. Nothing else is trustworthy.

## Raw file descriptors

`open`, `read`, `write`, `lseek` and `close` are the file API without stdio
in the way. A file descriptor is a small non-negative integer indexing into
the kernel's per-process table; 0, 1 and 2 are already taken by stdin,
stdout and stderr, so the first file you open is almost always 3.

```c
// rawio.c -- open/write/lseek/read with no stdio in sight
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>

int main(void) {
    int fd = open("demo.txt", O_RDWR | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) { perror("open"); return 1; }
    printf("open() returned fd %d\n", fd);

    const char *msg = "system calls are not function calls\n";
    ssize_t w = write(fd, msg, strlen(msg));
    printf("write() wrote %zd of %zu bytes\n", w, strlen(msg));

    off_t pos = lseek(fd, 0, SEEK_CUR);
    printf("file offset after write: %lld\n", (long long)pos);

    lseek(fd, 7, SEEK_SET);
    char buf[16];
    ssize_t r = read(fd, buf, 5);
    buf[r] = '\0';                       // read does not terminate anything
    printf("read 5 bytes at offset 7: \"%s\"\n", buf);

    off_t end = lseek(fd, 0, SEEK_END);
    printf("file size via lseek(SEEK_END): %lld\n", (long long)end);

    if (close(fd) < 0) perror("close");

    ssize_t bad = read(fd, buf, 5);      // deliberately use it after closing
    printf("read on closed fd -> %zd, errno %d (%s)\n", bad, errno, strerror(errno));
    return 0;
}
```

```text
open() returned fd 3
write() wrote 36 of 36 bytes
file offset after write: 36
read 5 bytes at offset 7: "calls"
file size via lseek(SEEK_END): 36
read on closed fd -> -1, errno 9 (Bad file descriptor)
```

Points worth extracting from those six lines:

- The third argument to `open` (`0644`) is the permission mode, and it is
  only consulted when `O_CREAT` actually creates the file. Omitting it while
  passing `O_CREAT` reads whatever garbage is next on the stack as the mode.
- Each fd carries **one shared offset**. `write` advanced it to 36;
  `lseek(fd, 7, SEEK_SET)` moved it back, so the subsequent `read` picked up
  mid-word. `lseek(fd, 0, SEEK_END)` is the classic way to get a file's size
  without a separate `stat`.
- `read` returning fewer bytes than requested is normal, not an error, and
  `0` means end-of-file. Anything that must read exactly *n* bytes needs a
  loop — the same discipline as `recv` in [the networking
  module](08-networking-basics.md).
- `errno 9 / EBADF` only became meaningful *after* the return value was
  seen to be -1. `errno` is never cleared on success, so reading it without
  first checking the return value gives you the error from some unrelated
  call ten functions ago.

## `errno`, `perror`, and what a failure actually says

`errno` is a per-thread modifiable lvalue (often a macro expanding to
`*__errno_location()`), so it is thread-safe, but it is only valid
immediately after a failed call — even `printf` can overwrite it. Capture it
into a local if you need it later.

`stat` is the natural place to see this, since one function has to cope with
files, directories and paths that do not exist:

```c
// statdemo.c -- ask the filesystem about a path, and handle errno properly
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <sys/stat.h>

static const char *kind(mode_t m) {
    if (S_ISREG(m))  return "regular file";
    if (S_ISDIR(m))  return "directory";
    if (S_ISLNK(m))  return "symlink";
    if (S_ISFIFO(m)) return "fifo";
    return "other";
}

static void report(const char *path) {
    struct stat st;
    if (stat(path, &st) < 0) {
        printf("%-12s ERROR: %s (errno %d)\n", path, strerror(errno), errno);
        return;
    }
    printf("%-12s %-13s %6lld bytes  mode %04o  links %u\n",
           path, kind(st.st_mode), (long long)st.st_size,
           st.st_mode & 07777, (unsigned)st.st_nlink);
}

int main(void) {
    report("demo.txt");
    report(".");
    report("/etc/hosts");
    report("/no/such/path");
    return 0;
}
```

```text
demo.txt     regular file      36 bytes  mode 0644  links 1
.            directory        352 bytes  mode 0755  links 11
/etc/hosts   regular file     213 bytes  mode 0644  links 1
/no/such/path ERROR: No such file or directory (errno 2)
```

`st_mode` packs the file type and the permission bits into one integer, so
you mask with `07777` to print permissions and use the `S_IS*` macros for
the type — never compare `st_mode` directly. Note also that `stat` follows
symlinks; `lstat` reports on the link itself, which is why `S_ISLNK` can
only ever be true for `lstat`.

## Processes: `fork`, `exec`, `wait`

`fork` is the strangest call in Unix: it returns **twice**. The child gets
`0`; the parent gets the child's pid. Both continue from the same line, with
identical copies of memory that then diverge.

```c
// forkdemo.c -- fork returns twice; exec never returns
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    printf("parent pid %d starting\n", getpid());
    fflush(stdout);                 // flush BEFORE forking -- see below

    pid_t pid = fork();
    if (pid < 0) { perror("fork"); return 1; }

    if (pid == 0) {
        printf("  child: fork() returned %d, my pid is %d\n", pid, getpid());
        fflush(stdout);
        execlp("echo", "echo", "  child: replaced by /bin/echo", (char *)NULL);
        perror("execlp");           // only reached if exec FAILED
        _exit(127);
    }

    printf("parent: fork() returned %d (the child's pid)\n", pid);
    fflush(stdout);

    int status;
    pid_t done = waitpid(pid, &status, 0);
    if (WIFEXITED(status))
        printf("parent: child %d exited with status %d\n", done, WEXITSTATUS(status));
    else if (WIFSIGNALED(status))
        printf("parent: child %d killed by signal %d\n", done, WTERMSIG(status));
    return 0;
}
```

```text
parent pid 16374 starting
parent: fork() returned 16375 (the child's pid)
  child: fork() returned 0, my pid is 16375
  child: replaced by /bin/echo
parent: child 16375 exited with status 0
```

The parent's line printed before the child's even though the child branch
appears first in the source — after `fork` the two processes are scheduled
independently and *nothing* orders their output. If you need ordering, you
need `waitpid` or a pipe, not luck.

`execlp` replaced the child's entire program image: same pid, completely
different code. Because it does not return on success, any line after it is
by definition the error path. And the `(char *)NULL` terminator must be
cast — passing a bare `NULL` to a variadic function can push the wrong
number of bytes and walk off the end of the argument list.

`waitpid` does two jobs: it blocks until the child finishes, and it reaps
the child's entry from the process table. Skip it and you accumulate
**zombies** — dead processes still occupying a table slot. The status is a
packed integer, so `WEXITSTATUS(status)` is required; comparing `status` to
`0` directly works by accident for success and misleads for everything else.

## Pipes: wiring one process's output to another's input

`pipe()` fills a two-element array with a read end and a write end. Combined
with `fork` and `dup2`, that is exactly how a shell implements `|`.

```c
// pipedemo.c -- build "wc -c" as a child whose stdin is our pipe
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    int fds[2];
    if (pipe(fds) < 0) { perror("pipe"); return 1; }
    printf("pipe: read end fd %d, write end fd %d\n", fds[0], fds[1]);
    fflush(stdout);

    pid_t pid = fork();
    if (pid == 0) {
        close(fds[1]);              // child never writes
        dup2(fds[0], STDIN_FILENO); // pipe read end becomes stdin
        close(fds[0]);
        execlp("wc", "wc", "-c", (char *)NULL);
        _exit(127);
    }

    close(fds[0]);                  // parent never reads
    const char *data = "twenty-nine bytes of text\nok\n";
    write(fds[1], data, strlen(data));
    close(fds[1]);                  // EOF for the child -- essential
    ...
}
```

```text
pipe: read end fd 3, write end fd 4
      29
child exited 0
```

`dup2(fds[0], STDIN_FILENO)` closes fd 0 and makes it a duplicate of the
pipe's read end, so `wc` — which knows nothing about pipes — just reads
"stdin" as usual. Then both processes close the end they do not use, which
is not tidiness but correctness: a reader only sees EOF when the **last**
open write end closes. Leave `fds[1]` open in the parent and `wc` blocks
forever waiting for input that will never come. That hang is one of the most
common bugs in hand-rolled pipelines.

## The buffering trap: stdio and syscalls do not mix freely

`write` goes straight to the kernel. `printf` fills a userspace buffer that
gets flushed later — at a newline if stdout is a terminal, only when full
(or at exit) if stdout is a file or pipe. `fork` copies that buffer, unflushed
contents and all.

```c
// bufferfork.c -- unflushed stdio buffers are duplicated by fork
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    printf("printf: buffered line\n");        // no fflush!
    write(STDOUT_FILENO, "write:  unbuffered line\n", 24);

    if (fork() == 0) {
        exit(0);            // exit() flushes the inherited copy of the buffer
    }
    wait(NULL);
    return 0;
}
```

Run once with output going to a terminal, and once redirected to a file:

```text
$ ./bufferfork                    # terminal: line-buffered
printf: buffered line
write:  unbuffered line

$ ./bufferfork > out.txt ; cat out.txt
write:  unbuffered line
printf: buffered line
printf: buffered line
```

Same binary, same source, two different outputs — and the redirected run
prints a line the program only executed once. To a terminal, the newline
flushed the buffer before `fork`, so ordering held and there was one copy.
Redirected, the buffer was still full at `fork` time, both processes
inherited it, and both flushed it at exit. This is why `forkdemo.c` calls
`fflush(stdout)` before forking, and why the child uses `_exit` (which skips
stdio flushing) rather than `exit` on the exec-failure path.

## Cheat sheet

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

## Exercise

Write `runpipe.c`, a program that implements `command1 | command2` from
`argv`, invoked as `./runpipe ls -l ~ wc -l` where the split point is a
literal `--` separator you choose. It must: create a pipe, fork twice, wire
the first child's stdout to the write end and the second child's stdin to
the read end with `dup2`, close **all four** parent-held ends, `exec` both
commands, and `waitpid` for both — reporting each exit status decoded with
`WIFEXITED`/`WEXITSTATUS`.

Then break it on purpose three ways and record what actually happens each
time: (1) skip closing the parent's copy of the write end and watch the
second command hang instead of finishing; (2) drop the `waitpid` calls, add
a `sleep(30)` at the end, and find your zombies with `ps` in another
terminal; (3) put an unflushed `printf` before the forks and redirect the
program's output to a file, then count how many times that one line appears.
