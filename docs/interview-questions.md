---
title: "C Interview Questions and Answers (180 Questions)"
description: "180 C interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the C Mastery Path."
---

# C Interview Questions and Answers

**180 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free C Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; First Program](level-1/01-setup.md)

**Q1.** What does `gcc hello.c -o hello` do?

**Answer:** Compiles hello.c and names the resulting binary hello

-o names the output; without it gcc produces a.out.

**Q2.** In `int main(void)`, what do `int` and `void` mean?

**Answer:** int is the exit status returned to the OS; void means main takes no arguments

A return of 0 signals success.

**Q3.** What does `#include <stdio.h>` do?

**Answer:** The preprocessor pastes in declarations for standard I/O functions like printf before compilation

It's a preprocessor directive, handled before the compiler proper.

**Q4.** On macOS, `gcc --version` prints 'Apple clang'. Is something broken?

**Answer:** No: gcc is usually an alias for clang there; both accept the options used in the course

Homebrew installs a real gcc if you want GNU extensions.

### [02 · Variables, Data Types &amp; Operators](level-1/02-variables-data-types.md)

**Q5.** What does this print?

```c
#include <stdio.h>
int main(void){ int a=7, b=2; double d=a/b; printf("%d %.1f", a/b, d); return 0; }
```

**Answer:** 3 3.0

Integer division truncates before the result is stored in a double. Verified by compiling and running.

**Q6.** Why use `sizeof(int)` rather than assuming 4 bytes?

**Answer:** The C standard doesn't fix type sizes; they depend on platform and compiler

sizeof is evaluated at compile time.

**Q7.** What happens if you read an uninitialized local `int score;`?

**Answer:** Undefined behavior: it holds garbage and the compiler doesn't zero it

Always initialize before use.

**Q8.** Which format specifier prints a `size_t` such as the result of sizeof?

**Answer:** %zu

size_t is unsigned and platform-sized.

### [03 · Control Flow](level-1/03-control-flow.md)

**Q9.** What does this print?

```c
#include <stdio.h>
int main(void){ int n=1; switch(n){ case 1: printf("one "); case 2: printf("two "); break; case 3: printf("three"); } return 0; }
```

**Answer:** one two

Without break, execution falls through into the next case. Verified by compiling and running.

**Q10.** What does this print?

```c
#include <stdio.h>
int main(void){ int a=10,b=3; int max=(a>b)?a:b; printf("%d", max); return 0; }
```

**Answer:** 10

The ternary operator produces a value. Verified by compiling and running.

**Q11.** In C before C99, how is truth represented?

**Answer:** Any non-zero value is true and 0 is false; C99 added _Bool via &lt;stdbool.h&gt;

Conditions test for non-zero.

**Q12.** What does this print?

```c
#include <stdio.h>
int main(void){ int x=0; if (x = 5) printf("yes"); else printf("no"); return 0; }
```

**Answer:** yes

= assigns 5, and the value 5 is non-zero so the branch runs. Use == to compare. Verified by compiling and running.

### [04 · Functions](level-1/04-functions.md)

**Q13.** What does this print?

```c
#include <stdio.h>
void inc(int n){ n = n + 1; }
int main(void){ int x=5; inc(x); printf("%d", x); return 0; }
```

**Answer:** 5

C passes by value, so the function changes only its own copy. Verified by compiling and running.

**Q14.** What is a function prototype for?

**Answer:** It tells the compiler the name, return type and parameter types so calls can be checked before the definition

Headers hold prototypes for sharing across files.

**Q15.** What does this print?

```c
#include <stdio.h>
double avg(int a,int b,int c){ return (a+b+c)/3.0; }
int main(void){ printf("%.2f", avg(4,7,9)); return 0; }
```

**Answer:** 6.67

Dividing by 3.0 forces floating-point division. Verified by compiling and running.

**Q16.** How can a function modify the caller's variable?

**Answer:** Pass its address and dereference the pointer inside the function

Pass-by-value copies the pointer, but it still points at the original.

### [05 · Arrays &amp; Strings](level-1/05-arrays-strings.md)

**Q17.** What does this print?

```c
#include <stdio.h>
#include <string.h>
int main(void){ char name[20]="Ada"; printf("%zu %zu", strlen(name), sizeof(name)); return 0; }
```

**Answer:** 3 20

strlen counts characters before '\0'; sizeof is the buffer capacity. Verified by compiling and running.

**Q18.** What does C do when you read `scores[5]` of a 5-element array?

**Answer:** Nothing is checked: it compiles and invokes undefined behavior, possibly corrupting memory

Track sizes yourself.

**Q19.** Why are `strcpy` and `strcat` risky?

**Answer:** They don't check that the destination buffer is large enough, so they can overflow it

Prefer bounded versions and generous buffers.

**Q20.** What does this print?

```c
#include <stdio.h>
#include <string.h>
int main(void){ printf("%d %d", strcmp("abc","abc")==0, "abc"=="abc"?7:7); return 0; }
```

**Answer:** 1 7

strcmp returns 0 when the strings are equal; never compare strings with ==. Verified by compiling and running.

### [06 · Pointers Basics](level-1/06-pointers-basics.md)

**Q21.** What does this print?

```c
#include <stdio.h>
int main(void){ int age=30; int *p=&age; *p=31; printf("%d", age); return 0; }
```

**Answer:** 31

*p = 31 writes through the pointer to age itself. Verified by compiling and running.

**Q22.** What do `&x` and `*p` mean in expressions?

**Answer:** &amp;x is the address of x; *p dereferences p to reach the value it points at

In a declaration, `int *p` means p is a pointer to int.

**Q23.** Why initialize a pointer to NULL when it has nothing to point at?

**Answer:** An uninitialized pointer holds a garbage address, and dereferencing it is undefined behavior

You can test `if (p != NULL)` before using it.

**Q24.** How do you print a pointer's address with printf?

**Answer:** %p with a (void *) cast

The cast is the portable convention.

### [07 · Structs](level-1/07-structs.md)

**Q25.** What does this print?

```c
#include <stdio.h>
struct P{int x,y;};
int main(void){ struct P a={10,20}; struct P b=a; b.x=99; printf("%d %d", a.x, b.x); return 0; }
```

**Answer:** 10 99

Assigning a struct copies all members, so changing b leaves a untouched. Verified by compiling and running.

**Q26.** Why doesn't `s.name = "Maria";` compile when name is a char array?

**Answer:** Arrays can't be assigned; use strcpy (or strncpy) to copy the characters in

Initialization at declaration is allowed; later assignment is not.

**Q27.** Does the declaration `struct Point { int x; int y; };` allocate memory?

**Answer:** No: it only describes the shape; `struct Point p;` creates a variable

It's like defining a type.

**Q28.** Can you compare two structs with `==`?

**Answer:** No: compare member by member

C has no built-in whole-struct equality.

### [08 · File I/O](level-1/08-file-io.md)

**Q29.** Which fopen mode erases an existing file's content as soon as it opens?

**Answer:** "w" (and "w+")

Use "a" to append instead.

**Q30.** What should you always do right after calling fopen?

**Answer:** Check whether it returned NULL before using the FILE pointer

A missing file, bad path or permissions all give NULL.

**Q31.** What does fopen("data.txt", "r") return if the file doesn't exist?

**Answer:** NULL

"r" never creates files; "w" and "a" do.

**Q32.** What does fprintf do differently from printf?

**Answer:** Its first argument is a FILE * saying where the formatted output goes

Otherwise it formats the same way.

### [09 · Preprocessor &amp; Multi-file Compilation](level-1/09-preprocessor-multifile.md)

**Q33.** What does this print?

```c
#include <stdio.h>
#define SQ(x) x*x
int main(void){ printf("%d", SQ(1+2)); return 0; }
```

**Answer:** 5

SQ(1+2) expands to 1+2*1+2 = 5: macros are pure text substitution, so wrap parameters in parentheses. Verified by compiling and running.

**Q34.** What is a header guard for?

**Answer:** Making a second #include of the same header a no-op, avoiding redefinition errors

#ifndef NAME / #define NAME / #endif.

**Q35.** What's the difference between #include &lt;x.h&gt; and #include "x.h"?

**Answer:** Angle brackets search system paths; quotes search the including file's directory first

Use quotes for your own project headers.

**Q36.** What happens during preprocessing?

**Answer:** Lines starting with # are handled: includes pasted, macros expanded, before the compiler sees real C

It's pure text processing.

### [10 · Project — CLI Contact Book](level-1/10-project-contact-book.md)

**Q37.** Why does add_contact copy with strncpy and then set the last byte to '\0'?

**Answer:** strncpy may not null-terminate when the source fills the buffer, so the terminator is forced

It also prevents overflowing name[NAME_LEN].

**Q38.** Why does add_contact take `int *count`?

**Answer:** So it can increment the caller's count through the pointer after adding

Pass-by-value would only change a copy.

**Q39.** What does load_contacts return when contacts.txt does not exist yet?

**Answer:** 0: the book simply starts empty

It treats a missing file as no contacts.

**Q40.** What does save_contacts' "w" mode mean for the file?

**Answer:** It rewrites the whole file from the in-memory array each time

The array is the source of truth.

## Level 2 · Intermediate

### [01 · Pointers Deep Dive](level-2/01-pointers-deep-dive.md)

**Q41.** What does this print?

```c
#include <stdio.h>
int main(void){ int a[]={10,20,30,40}; int *p=a; printf("%d %d %d", *(p+2), p[1], (int)((a+3)-p)); return 0; }
```

**Answer:** 30 20 3

p+n advances n elements, and subtracting pointers counts elements, not bytes. Verified by compiling and running.

**Q42.** What does this print?

```c
#include <stdio.h>
int main(void){ int a[]={1,2,3}; int *p=a; int x=*p++; printf("%d %d", x, *p); return 0; }
```

**Answer:** 1 2

Postfix ++ binds tighter than *: read *p, then advance p. Verified by compiling and running.

**Q43.** What does `ip + 1` do when ip is an `int *`?

**Answer:** Advances the pointer by sizeof(int) bytes, one whole element

Pointer arithmetic is scaled by the pointed-to type.

**Q44.** Why can qsort sort any element type?

**Answer:** It takes a void * array, an element size and a comparison function pointer

Function pointers plus void * give C its generic algorithms.

### [02 · Dynamic Memory](level-2/02-dynamic-memory.md)

**Q45.** Why write `malloc(n * sizeof *nums)` instead of `sizeof(int)`?

**Answer:** It stays correct if nums' type changes, since the size is derived from the pointer

malloc takes a size in bytes.

**Q46.** What must you do after calling malloc?

**Answer:** Check for NULL, since the allocation can fail

Also free the block when finished.

**Q47.** What are the two most expensive bug classes in this module?

**Answer:** Memory leaks and use-after-free

There's no garbage collector.

**Q48.** After free(p), what is good practice?

**Answer:** Set p = NULL so the dangling pointer is harmless

Double free and use-after-free are undefined behavior.

### [03 · Multi-dimensional Arrays &amp; String Manipulation](level-2/03-multidim-arrays-strings.md)

**Q49.** What does this print?

```c
#include <stdio.h>
int main(void){ int g[3][4]={{1,2,3,4},{5,6,7,8},{9,10,11,12}}; int *f=&g[0][0]; printf("%d %zu %zu", f[6], sizeof g, sizeof g[0]); return 0; }
```

**Answer:** 7 48 16

A 2-D array is one flat row-major block; sizes assume 4-byte int. Verified by compiling and running.

**Q50.** Why must the column count appear in a parameter like `int grid[][COLS]`?

**Answer:** The compiler needs it to compute row offsets (r * COLS + c)

A 2-D array decays to a pointer to its first row: int (*)[COLS].

**Q51.** Which loop order is cache-friendly for a row-major 2-D array?

**Answer:** Rows in the outer loop, columns in the inner loop

It walks memory sequentially.

**Q52.** Which function is dangerous because it cannot limit how much it writes?

**Answer:** strcpy (and gets, strcat): no destination size is given

Use bounded alternatives such as snprintf.

### [04 · Structs &amp; Unions Advanced](level-2/04-structs-unions-advanced.md)

**Q53.** What does this print (typical 64-bit)?

```c
#include <stdio.h>
struct Bad{char a;int b;char c;};
struct Good{int b;char a;char c;};
int main(void){ printf("%zu %zu", sizeof(struct Bad), sizeof(struct Good)); return 0; }
```

**Answer:** 12 8

Padding aligns each field; ordering large members first wastes less. Verified by compiling and running.

**Q54.** What does `p->quantity` mean?

**Answer:** The same as (*p).quantity: the member of the struct p points to

Passing a pointer copies 8 bytes instead of the whole struct and lets the function modify the original.

**Q55.** What is a union?

**Answer:** A type whose members share the same memory, so only one is meaningful at a time

Its size is that of its largest member.

**Q56.** Why mark a parameter `const Item *`?

**Answer:** It promises the function won't modify the caller's struct, and the compiler enforces it

A non-const pointer is for functions that modify.

### [05 · Binary File I/O](level-2/05-binary-file-io.md)

**Q57.** What do fread and fwrite return?

**Answer:** The number of items (not bytes) transferred; a short count means EOF or error

size * count bytes are moved.

**Q58.** Why include `b` in the mode ("rb", "wb")?

**Answer:** On Windows it stops \n translation to \r\n that would corrupt binary data; it costs nothing elsewhere

Always include it for portability.

**Q59.** Why is binary I/O good for random access?

**Answer:** Fixed-size records let you jump straight to record N with fseek

Text lines are variable-width.

**Q60.** Why is a binary file not automatically portable across machines?

**Answer:** The bytes mirror the in-memory representation (endianness, type sizes, padding)

Define the format carefully if sharing.

### [06 · Error Handling Conventions](level-2/06-error-handling.md)

**Q61.** What does this print?

```c
#include <stdio.h>
#include <stdlib.h>
int main(void){ printf("%d %ld", atoi("abc"), strtol("12x",NULL,10)); return 0; }
```

**Answer:** 0 12

atoi can't tell 'abc' from '0'; strtol with an end pointer lets you detect trailing junk. Verified by compiling and running.

**Q62.** Why return a status and pass the value through an out-parameter?

**Answer:** Any return value could be a valid result, so a sentinel like -1 is ambiguous

This is the most common design in real C APIs.

**Q63.** What does fopen return on failure besides NULL?

**Answer:** It also sets errno, which perror or strerror can describe

Read errno immediately; other calls can overwrite it.

**Q64.** Why is atoi unsafe for input you don't control?

**Answer:** It returns 0 for invalid input, indistinguishable from a real 0, and has no error reporting

Use strtol.

### [07 · Modular Programming](level-2/07-modular-programming.md)

**Q65.** What does file-scope `static` do to a function?

**Answer:** Makes it private to that translation unit, invisible to the linker and other files

It's the module's privacy keyword.

**Q66.** Why shouldn't you define a variable in a header?

**Answer:** Every .c that includes it gets its own copy and the linker reports duplicate symbols; use extern in the header and define once in a .c

Include only what callers need to compile against.

**Q67.** What belongs in a header?

**Answer:** Public prototypes, typedefs/enums callers need and the include guard

Implementation stays in the .c.

**Q68.** What is an opaque type?

**Answer:** A struct declared in the header but defined only in the .c, so users hold pointers and can't touch fields

It hides internals behind accessor functions.

### [08 · Makefiles &amp; Build Systems](level-2/08-makefiles-build-systems.md)

**Q69.** What is the most common Makefile error and its message?

**Answer:** Indenting a recipe with spaces instead of a TAB: 'missing separator'

The message doesn't mention tabs.

**Q70.** How does make decide to rebuild a target?

**Answer:** If any prerequisite is newer than the target (timestamps)

Hence 'app is up to date' when nothing changed.

**Q71.** Which assignment operator lets users override with `make CC=clang`?

**Answer:** ?= (set only if not already defined)

:= is evaluated once immediately and is the default choice.

**Q72.** Why add `-MMD -MP` to a Makefile?

**Answer:** To auto-generate header dependency files so changing a .h rebuilds the files that include it

It keeps dependency tracking correct as the project grows.

### [09 · Debugging Tools](level-2/09-debugging-tools.md)

**Q73.** Why build with `-g -O0` for debugging?

**Answer:** -g embeds symbols; -O0 stops the optimizer from reordering or removing variables

Optimize only for release.

**Q74.** In gdb, what's the difference between `next` and `step`?

**Answer:** next steps over function calls; step steps into them

Use continue to run to the next breakpoint.

**Q75.** What does valgrind catch that a normal run may not?

**Answer:** Memory bugs like leaks, invalid reads/writes and uninitialized use, even if they don't crash today

Undefined behavior can look fine until production.

**Q76.** The sum_range(0, 5) bug reads values[5]. Why might it appear to work?

**Answer:** Out-of-bounds read is undefined behavior, not a guaranteed crash

This is exactly the bug a debugger or valgrind is for.

### [10 · Project — Linked-list Inventory System](level-2/10-project-inventory-system.md)

**Q77.** Why use a linked list for the inventory instead of an array?

**Answer:** Adding an item never needs to resize or copy existing data

Trade-off: pointer chasing and per-node allocation.

**Q78.** How does the project persist the list to a binary file?

**Answer:** It flattens the list to an array of Items on save and rebuilds nodes on load

Pointers aren't meaningful after the process exits.

**Q79.** What must callers never do with the pointer returned by inventory_find?

**Answer:** Free it: the inventory owns the nodes

It points into the list.

**Q80.** What does adjusting stock refuse to do?

**Answer:** Take a quantity below zero, returning 0 and leaving the item unchanged

It validates before mutating.

## Level 3 · Advanced

### [01 · Data Structures in C](level-3/01-data-structures.md)

**Q81.** What does this print?

```c
#include <stdio.h>
#include <stdlib.h>
typedef struct N{int v;struct N*next;}N;
int main(void){ N*h=NULL; for(int i=1;i<=3;i++){N*n=malloc(sizeof*n);n->v=i;n->next=h;h=n;} for(N*p=h;p;p=p->next)printf("%d ",p->v); return 0; }
```

**Answer:** 3 2 1

Each new node is inserted at the head, so the list reads in reverse insertion order. Verified by compiling and running.

**Q82.** What is the time to insert at the head of a singly linked list?

**Answer:** O(1): allocate a node and relink the head

Appending at the tail needs a tail pointer or O(n) walk.

**Q83.** Why free a list with a loop that saves `next` first?

**Answer:** Reading n-&gt;next after free(n) is use-after-free

Save the successor before freeing the node.

**Q84.** What does a stack built on an array need to guard against?

**Answer:** Overflow when pushing to a full array and underflow when popping an empty one

Return a status instead of corrupting memory.

### [02 · Trees &amp; Basic Graph Representations](level-3/02-trees-graphs.md)

**Q85.** What does an in-order walk of this BST print?

```c
#include <stdio.h>
#include <stdlib.h>
typedef struct T{int k;struct T*l,*r;}T;
T*ins(T*t,int k){ if(!t){t=calloc(1,sizeof*t);t->k=k;return t;} if(k<t->k)t->l=ins(t->l,k); else t->r=ins(t->r,k); return t;}
void in(T*t){ if(!t)return; in(t->l); printf("%d ",t->k); in(t->r);}
int main(void){ T*r=NULL; int a[]={5,2,8,1,9,3}; for(int i=0;i<6;i++)r=ins(r,a[i]); in(r); return 0; }
```

**Answer:** 1 2 3 5 8 9

In-order traversal of a BST yields keys in sorted order. Verified by compiling and running.

**Q86.** What is the BST invariant?

**Answer:** Every left descendant is smaller and every right descendant larger than the node

It enables O(log n) search when balanced.

**Q87.** When does a plain BST degrade to O(n) lookups?

**Answer:** When keys are inserted in sorted order, producing a chain

Self-balancing trees fix this.

**Q88.** Which traversal uses a queue?

**Answer:** Breadth-first search (level by level)

DFS uses the call stack or an explicit stack.

### [03 · Sorting &amp; Searching Algorithms](level-3/03-sorting-searching.md)

**Q89.** What does this print?

```c
#include <stdio.h>
#include <stdlib.h>
int cmp(const void*a,const void*b){ int x=*(const int*)a,y=*(const int*)b; return (x>y)-(x<y);}
int main(void){ int a[]={5,2,9,1}; qsort(a,4,sizeof a[0],cmp); int k=2; int*f=bsearch(&k,a,4,sizeof a[0],cmp); printf("%d %d %d %d %d",a[0],a[1],a[2],a[3],(int)(f-a)); return 0; }
```

**Answer:** 1 2 5 9 1

qsort sorts in place; bsearch needs sorted data and returns a pointer to the match. Verified by compiling and running.

**Q90.** What is quicksort's worst-case time?

**Answer:** O(n²), e.g. always picking a bad pivot on already-sorted data without safeguards

A good pivot choice gives O(n log n) on average.

**Q91.** What precondition does binary search need?

**Answer:** The array must be sorted

Each step halves the range: O(log n).

**Q92.** Why subtract with comparisons like `(x>y)-(x<y)` rather than `x-y` in a comparator?

**Answer:** x-y can overflow for large ints, giving a wrong sign

Overflow is undefined behavior for signed ints.

### [04 · Bit Manipulation](level-3/04-bit-manipulation.md)

**Q93.** What does this print?

```c
#include <stdio.h>
int main(void){ unsigned f=0; f|=1u<<2; f|=1u<<0; f&=~(1u<<0); f^=1u<<3; printf("%u %d", f, (f>>2)&1); return 0; }
```

**Answer:** 12 1

Set bit2, set bit0, clear bit0, toggle bit3: 4+8 = 12, and bit 2 is set. Verified by compiling and running.

**Q94.** How do you clear bit n of x?

**Answer:** x &amp;= ~(1u &lt;&lt; n)

| sets, ^ toggles, &amp; with the inverted mask clears.

**Q95.** Why prefer unsigned types for bit manipulation?

**Answer:** Shifting and right-shifting signed negative values has implementation-defined or undefined behavior

Unsigned semantics are fully defined.

**Q96.** What does this print?

```c
#include <stdio.h>
int main(void){ unsigned n=0xB5; int c=0; while(n){ n&=n-1; c++; } printf("%d",c); return 0; }
```

**Answer:** 5

n &amp; (n-1) clears the lowest set bit; 0xB5 = 10110101 has five bits set. Verified by compiling and running.

### [05 · Memory Management Deep Dive](level-3/05-memory-management-deep-dive.md)

**Q97.** Where do local variables, malloc'd blocks, globals and code live?

**Answer:** Stack, heap, data/BSS, and text segments respectively

Four regions in one address space.

**Q98.** Which bug does this describe: freeing a block and then reading it?

**Answer:** Use-after-free (a dangling pointer)

Set pointers to NULL after free.

**Q99.** What does valgrind/ASan report for malloc without free before exit?

**Answer:** A memory leak: the block was still reachable or lost at exit

Leaks matter most in long-running programs.

**Q100.** What does realloc return when it can't grow a block?

**Answer:** NULL, leaving the original block valid; assign to a temp pointer first

`p = realloc(p, n)` leaks the original on failure.

### [06 · Static &amp; Dynamic Libraries](level-3/06-static-dynamic-libraries.md)

**Q101.** What is a static library (.a)?

**Answer:** An archive of object files copied into the executable at link time

The binary is self-contained but bigger.

**Q102.** What is a shared library (.so/.dylib)?

**Answer:** Code loaded at run time and shared between processes; the executable records the dependency

Updating the library updates all users, but it must be findable at run time.

**Q103.** Which tool creates a static library from .o files?

**Answer:** ar (e.g. ar rcs libx.a a.o b.o)

Link with -L. -lx.

**Q104.** Why compile shared-library objects with `-fPIC`?

**Answer:** Position-independent code can be loaded at any address

Needed for .so building.

### [07 · Concurrency Basics](level-3/07-concurrency-basics.md)

**Q105.** What does this print?

```c
#include <stdio.h>
#include <pthread.h>
void*w(void*a){ *(int*)a*=2; return NULL; }
int main(void){ int x=21; pthread_t t; pthread_create(&t,NULL,w,&x); pthread_join(t,NULL); printf("%d",x); return 0; }
```

**Answer:** 42

join waits for the thread to finish, so the write is visible afterwards. Verified by compiling and running.

**Q106.** What is a data race?

**Answer:** Two threads accessing the same memory concurrently with at least one write and no synchronization

The result is undefined behavior.

**Q107.** What does pthread_join do?

**Answer:** Blocks until the thread finishes and optionally collects its return value

Always join or detach threads.

**Q108.** What does a mutex protect?

**Answer:** A critical section so only one thread at a time executes it

Lock before, unlock after, on every path.

### [08 · Networking Basics](level-3/08-networking-basics.md)

**Q109.** What is the order of calls that make a TCP server?

**Answer:** socket, bind, listen, accept

Then read/write on the accepted socket.

**Q110.** What does accept() return?

**Answer:** A new socket descriptor for that client connection

The listening socket keeps accepting.

**Q111.** Why must you check the return of read/recv?

**Answer:** It may return fewer bytes than requested, 0 on close, or -1 on error

TCP is a byte stream, not messages.

**Q112.** Why use htons() for the port?

**Answer:** Network byte order is big-endian, so host values must be converted

Use ntohs on receive.

### [09 · Working with Unix System Calls](level-3/09-unix-system-calls.md)

**Q113.** What are file descriptors 0, 1 and 2?

**Answer:** stdin, stdout and stderr

open() returns the lowest unused descriptor.

**Q114.** What does fork() return in the child?

**Answer:** 0 (the parent gets the child's PID)

-1 means failure.

**Q115.** Why must a parent call wait/waitpid?

**Answer:** To collect the child's exit status and avoid leaving zombie processes

A zombie is an exited process whose status was never collected.

**Q116.** How do read/write differ from fread/fwrite?

**Answer:** They are unbuffered system calls on file descriptors; stdio adds user-space buffering

Buffered I/O can need fflush.

### [10 · Project — Multi-threaded Key-Value Store / Tiny HTTP Server](level-3/10-project-kv-store.md)

**Q117.** What does a key-value store's get(key) typically use for speed?

**Answer:** A hash table giving average O(1) lookup

A hash function maps keys to buckets.

**Q118.** How are hash collisions handled with chaining?

**Answer:** Each bucket holds a linked list of entries with the same hash

Open addressing is the alternative.

**Q119.** Why persist the store with a write-then-rename pattern?

**Answer:** So a crash mid-write can't leave a half-written data file

Rename is atomic on POSIX.

**Q120.** Why validate key and value lengths before copying?

**Answer:** To prevent buffer overflows in fixed-size fields

Never trust sizes from a file or the network.

## Level 4 · Master

### [01 · Advanced Memory Management](level-4/01-advanced-memory-management.md)

**Q121.** Why is an arena (bump) allocator fast?

**Answer:** Allocation is a pointer add, and everything is freed at once by resetting the pointer

No per-object free means no fragmentation bookkeeping.

**Q122.** What's the main restriction of an arena?

**Answer:** You can't free individual objects; lifetimes must be grouped

Great for request-scoped or parse-tree data.

**Q123.** What must a custom allocator respect for each returned pointer?

**Answer:** Alignment suitable for the object type

Misaligned access is slow or faulting on some CPUs.

**Q124.** What does a free list in a pool allocator store?

**Answer:** Fixed-size free slots linked together inside the free memory itself

Alloc pops, free pushes: O(1).

### [02 · Concurrency &amp; Synchronization](level-4/02-concurrency-synchronization.md)

**Q125.** Between a mutex, an atomic and no sync, which gives a correct shared counter with the least overhead?

**Answer:** An atomic increment (e.g. atomic_fetch_add) for a simple counter

The lesson measures all three; no-sync is fast but wrong.

**Q126.** What is a deadlock?

**Answer:** Threads wait forever for locks held by each other

Acquire locks in a consistent global order to avoid it.

**Q127.** What does a condition variable let a thread do?

**Answer:** Sleep until another thread signals that a condition may have changed

Always wait in a loop to recheck the predicate.

**Q128.** Why does an unsynchronized counter lose updates?

**Answer:** Increment is read-modify-write, so two threads can read the same old value

Lost updates give totals below the expected count.

### [03 · Performance Optimization &amp; Profiling](level-4/03-performance-profiling.md)

**Q129.** What is the first rule of optimization?

**Answer:** Measure first (honest timing/profiling) and optimize the hot spot, not guesses

Compile benchmarks with optimization enabled.

**Q130.** Why does row-major iteration of a 2-D array beat column-major?

**Answer:** It walks memory sequentially and uses CPU cache lines efficiently

Cache locality dominates for large matrices.

**Q131.** What does a profiler like gprof/perf show?

**Answer:** Where time is spent per function so you target the hottest code

Instrumented tools add overhead; sampling is cheaper.

**Q132.** What does llvm-cov report?

**Answer:** Which lines/branches executed, i.e. code coverage from instrumented runs

Useful for finding untested code.

### [04 · Writing Portable C](level-4/04-writing-portable-c.md)

**Q133.** What does the C standard guarantee about `int` size?

**Answer:** Only a minimum range (at least 16 bits); the exact size is implementation-defined

Use &lt;stdint.h&gt; for exact widths.

**Q134.** Which header gives int32_t and uint64_t?

**Answer:** &lt;stdint.h&gt;

Fixed-width types make sizes explicit.

**Q135.** Why is it a portability bug to write a struct to disk with fwrite and read it elsewhere?

**Answer:** Padding, type sizes and endianness differ across platforms

Serialize field by field in a defined byte order.

**Q136.** What does compiling with -std=c11 -Wall -Wextra -pedantic help find?

**Answer:** Non-standard extensions and suspicious constructs that break on other compilers

It keeps code portable.

### [05 · Security in C](level-4/05-security-in-c.md)

**Q137.** Why is a buffer overflow dangerous even when the program doesn't crash?

**Answer:** It silently overwrites adjacent memory, which an attacker can use to corrupt data or hijack control flow

The lesson's title point: the one that does not crash.

**Q138.** Which is the safer way to copy into a fixed buffer?

**Answer:** snprintf(buf, sizeof buf, "%s", src) (or another size-bounded call)

Always pass the destination size.

**Q139.** What's the format-string bug in `printf(user_input)`?

**Answer:** The user controls the format, so %s/%n can read or write memory

Use printf("%s", user_input).

**Q140.** Which compiler/runtime options help catch memory errors?

**Answer:** -fsanitize=address,undefined (AddressSanitizer/UBSan) in testing

Combine with stack protectors and FORTIFY_SOURCE.

### [06 · Advanced Data Structures](level-4/06-advanced-data-structures.md)

**Q141.** What invariant does an AVL tree keep?

**Answer:** The heights of every node's two subtrees differ by at most 1

Rotations restore balance after insert/delete.

**Q142.** Why is an AVL tree guaranteed O(log n)?

**Answer:** Balance limits its height to O(log n)

A plain BST can degrade to a chain.

**Q143.** How many distinct rotation cases does AVL insert have?

**Answer:** Four: LL, RR, LR and RL

Double rotations handle the zig-zag cases.

**Q144.** What problem does a skip list or hash table solve relative to a sorted array?

**Answer:** Fast insertion/lookup without O(n) shifting

The lesson quantifies the cost of the naive approach.

### [07 · Interfacing with Other Languages](level-4/07-interfacing-other-languages.md)

**Q145.** What does `extern "C"` do in a C++ header for a C library?

**Answer:** Prevents C++ name mangling so the C symbols link correctly

Needed to call C from C++.

**Q146.** Why do FFI-friendly C libraries use opaque handles and plain types?

**Answer:** Other languages can call them without knowing struct layouts or C++ features

A library designed to be called.

**Q147.** Who must free memory a C library allocated and returned to Python/Rust?

**Answer:** Whoever the API documents, normally via the library's own free function

Mixing allocators is undefined behavior.

**Q148.** What does an ABI define?

**Answer:** Binary-level conventions: calling convention, struct layout and symbol names

C has a stable de-facto ABI per platform.

### [08 · Testing in C](level-4/08-testing-in-c.md)

**Q149.** What is the core of a tiny C test framework?

**Answer:** A macro that checks a condition, reports file:line on failure and tracks a pass/fail count

Forty lines can be enough.

**Q150.** Why run tests under ASan/valgrind too?

**Answer:** Passing assertions can still hide memory errors

Sanitizers find leaks and overflows.

**Q151.** How should a test program signal failure to CI?

**Answer:** Return a non-zero exit status from main

CI treats non-zero as failed.

**Q152.** What makes a function easy to unit test?

**Answer:** Few hidden dependencies: pass inputs in, return outputs, avoid global state

Dependency injection via function pointers helps.

### [09 · Build Systems at Scale](level-4/09-build-systems-at-scale.md)

**Q153.** What does `make -j8` do?

**Answer:** Runs up to 8 recipes in parallel, which is safe only when dependencies are correct

Missing dependencies cause flaky parallel builds.

**Q154.** What is an out-of-source build?

**Answer:** Object files and binaries go in a separate build directory, keeping the source tree clean

It allows several configurations side by side.

**Q155.** Why generate dependency files (-MMD -MP)?

**Answer:** So make knows which headers each object depends on

Otherwise edited headers don't trigger rebuilds.

**Q156.** What do CMake and Meson do?

**Answer:** Generate build files (Makefiles/Ninja) from a higher-level description

They are meta-build systems.

### [10 · Capstone Project](level-4/10-capstone-project.md)

**Q157.** What is the point of the capstone's public API header?

**Answer:** A small stable contract so callers depend on function prototypes, not internals

It ties together modules, error handling and tests.

**Q158.** What should every function that can fail do?

**Answer:** Report the failure through a documented status or errno, and callers must check it

C has no exceptions.

**Q159.** Which tools would you combine to verify the capstone?

**Answer:** -Wall -Wextra, unit tests, ASan/UBSan and valgrind/gdb

Layered checks catch different bug classes.

**Q160.** What ties together ownership in the capstone's API?

**Answer:** Documented ownership rules: who allocates and who frees every pointer crossing the API

Unclear ownership is the root of leaks and double frees.

## Mixed questions across all levels

**Q161.** What does this print?

```c
#include <stdio.h>
int main(void){ int a=7,b=2; printf("%d %.1f %d", a/b, (double)a/b, a%b); return 0; }
```

**Answer:** 3 3.5 1

Integer division truncates; cast one operand to double for a fractional result. Verified by compiling and running.

**Q162.** What does this print?

```c
#include <stdio.h>
int main(void){ int n=2; switch(n){ case 1: printf("a"); case 2: printf("b"); case 3: printf("c"); break; default: printf("d"); } return 0; }
```

**Answer:** bc

Missing break falls through case 3 and stops at break. Verified by compiling and running.

**Q163.** What does this print?

```c
#include <stdio.h>
void f(int *p, int v){ *p = v; v = 99; }
int main(void){ int x=1,y=2; f(&x,y); printf("%d %d", x, y); return 0; }
```

**Answer:** 2 2

The pointer lets f change x; v is a copy, so y is untouched. Verified by compiling and running.

**Q164.** What does this print?

```c
#include <stdio.h>
#define DOUBLE(x) x*2
int main(void){ printf("%d", DOUBLE(3+4)); return 0; }
```

**Answer:** 11

The macro expands to 3+4*2, which is 11, not 14: wrap macro parameters in parentheses. Verified by compiling and running.

**Q165.** What does this print?

```c
#include <stdio.h>
int main(void){ int a[]={4,8,15,16}; int *p=a+1; printf("%d %d %d", *p, p[2], (int)(&a[3]-p)); return 0; }
```

**Answer:** 8 16 2

p points at a[1]; p[2] is a[3]; pointer difference counts elements. Verified by compiling and running.

**Q166.** What does this print (typical 64-bit)?

```c
#include <stdio.h>
typedef struct { char a; double b; char c; } S;
int main(void){ printf("%zu", sizeof(S)); return 0; }
```

**Answer:** 24

Padding aligns the double to 8 bytes: 1+7 pad, 8, 1+7 pad. Verified by compiling and running.

**Q167.** What does this print?

```c
#include <stdio.h>
#include <string.h>
int main(void){ char s[16]="hello"; printf("%zu %zu %c", strlen(s), sizeof s, s[strlen(s)]=='\0'?'Y':'N'); return 0; }
```

**Answer:** 5 16 Y

strlen stops at the terminator; sizeof is the whole buffer; the byte after the last char is '\0'. Verified by compiling and running.

**Q168.** What does this print?

```c
#include <stdio.h>
int main(void){ unsigned x=0x2C; printf("%u %u %u", x&0x0F, x|0x01, x>>2); return 0; }
```

**Answer:** 12 45 11

0x2C = 44: &amp; 0x0F gives 12, | 1 gives 45, &gt;&gt; 2 gives 11. Verified by compiling and running.

**Q169.** Why does `p = realloc(p, newsize)` risk a leak?

**Answer:** On failure realloc returns NULL, overwriting p and losing the original block; assign to a temporary first

Check the temp for NULL, then assign it to p.

**Q170.** A function returns a pointer to a local array. What's wrong?

**Answer:** The array lives on the stack and is gone when the function returns: a dangling pointer

Return heap memory, a static buffer, or write into a caller-provided buffer.

**Q171.** You free a node then read its `next`. What happened?

**Answer:** Use-after-free: save next before freeing

Undefined behavior; ASan reports it.

**Q172.** `make` rebuilds everything after you edit only a header. Which fix is correct?

**Answer:** Generate dependency files with -MMD -MP and include them so objects depend on the headers they use (and the opposite problem, stale builds, is also avoided)

Edited headers must invalidate dependent objects.

**Q173.** A server reads a request with a single recv() into a 64-byte buffer. What can go wrong?

**Answer:** TCP is a byte stream: the read may return a partial message, and unbounded copying could overflow the buffer; loop and bound it

Check the return value and size every copy.

**Q174.** Two threads increment a shared counter 1,000,000 times each with no synchronization. What do you expect?

**Answer:** A total below 2,000,000 because increments are read-modify-write and updates are lost; use an atomic or mutex

A data race is undefined behavior.

**Q175.** Which is the safest way to build a path from user input into a 64-byte buffer?

**Answer:** snprintf(buf, sizeof buf, "%s/%s", dir, name) and check that the result is less than sizeof buf

snprintf never writes past the size and reports truncation.

**Q176.** Why compile and test with -fsanitize=address,undefined?

**Answer:** They turn silent memory errors and undefined behavior into immediate, located reports

Use them in CI along with warnings -Wall -Wextra.

**Q177.** Which structure gives guaranteed O(log n) search with inserts and deletes?

**Answer:** A self-balancing tree such as an AVL tree

Rotations keep the height logarithmic.

**Q178.** You need many small short-lived allocations freed all at once (e.g. per request). Which allocator fits?

**Answer:** An arena: allocation is a pointer bump and reset frees everything

Lifetimes are grouped.

**Q179.** What should an exit code of a test runner convey to CI?

**Answer:** Non-zero means at least one test failed

CI gates on the exit status.

**Q180.** What is the safest design for a C function that can fail and returns a value?

**Answer:** Return a status code and deliver the value through an out-parameter, documenting ownership of any allocation

Sentinels can collide with valid values.

## Practical coding questions

### Reverse a linked list in place

Implement reverse(): re-point each node's next pointer so the list is reversed without allocating new nodes, and return the new head. The program pushes 1..4 at the head (so it prints 4 3 2 1) and then prints the reversed list.

Starter code:

```
#include <stdio.h>
#include <stdlib.h>
typedef struct Node { int v; struct Node *next; } Node;

Node *push(Node *head, int v) {
    Node *n = malloc(sizeof *n);
    n->v = v; n->next = head;
    return n;
}
Node *reverse(Node *head) {
    // reverse the list in place by re-pointing each next; return the new head
    return head;
}
void print(Node *h) { for (; h; h = h->next) printf("%d%c", h->v, h->next ? ' ' : '\n'); }
int main(void) {
    Node *h = NULL;
    for (int i = 1; i <= 4; i++) h = push(h, i);
    print(h);
    h = reverse(h);
    print(h);
    while (h) { Node *n = h->next; free(h); h = n; }
    return 0;
}
```

Expected output:

```text
4 3 2 1
1 2 3 4
```

??? success "Show a solution"

    ```
    #include <stdio.h>
    #include <stdlib.h>
    typedef struct Node { int v; struct Node *next; } Node;

    Node *push(Node *head, int v) {
        Node *n = malloc(sizeof *n);
        n->v = v; n->next = head;
        return n;
    }
    Node *reverse(Node *head) {
        Node *prev = NULL;
        while (head) {
            Node *next = head->next;
            head->next = prev;
            prev = head;
            head = next;
        }
        return prev;
    }
    void print(Node *h) { for (; h; h = h->next) printf("%d%c", h->v, h->next ? ' ' : '\n'); }
    int main(void) {
        Node *h = NULL;
        for (int i = 1; i <= 4; i++) h = push(h, i);
        print(h);
        h = reverse(h);
        print(h);
        while (h) { Node *n = h->next; free(h); h = n; }
        return 0;
    }
    ```

    Walk the list keeping prev, save next before overwriting, and return prev as the new head. It's O(n) time and O(1) extra space. Verified by compiling and running the solution with gcc.

### A bounded string copy

Implement safe_copy(dst, cap, src): copy src into dst without overflowing, always null-terminate when cap &gt; 0, and return 1 if everything fit or 0 if it was truncated. Print the result for a 6-byte and a 32-byte destination.

Starter code:

```
#include <stdio.h>
#include <string.h>

/* copy src into dst (capacity cap); return 1 if it all fit, 0 if truncated.
   dst is always null-terminated when cap > 0. */
int safe_copy(char *dst, size_t cap, const char *src) {
    // copy without overflowing; always terminate; return 1 if it fit, else 0
    return 0;
}
int main(void) {
    char small[6], big[32];
    int a = safe_copy(small, sizeof small, "Mastery");
    int b = safe_copy(big, sizeof big, "Mastery");
    printf("%d [%s]\n%d [%s]\n", a, small, b, big);
    return 0;
}
```

Expected output:

```text
0 [Maste]
1 [Mastery]
```

??? success "Show a solution"

    ```
    #include <stdio.h>
    #include <string.h>

    /* copy src into dst (capacity cap); return 1 if it all fit, 0 if truncated.
       dst is always null-terminated when cap > 0. */
    int safe_copy(char *dst, size_t cap, const char *src) {
        if (cap == 0) return 0;
        size_t n = strlen(src);
        if (n >= cap) {
            memcpy(dst, src, cap - 1);
            dst[cap - 1] = '\0';
            return 0;
        }
        memcpy(dst, src, n + 1);
        return 1;
    }
    int main(void) {
        char small[6], big[32];
        int a = safe_copy(small, sizeof small, "Mastery");
        int b = safe_copy(big, sizeof big, "Mastery");
        printf("%d [%s]\n%d [%s]\n", a, small, b, big);
        return 0;
    }
    ```

    Compare strlen(src) with cap: if it doesn't fit, copy cap-1 bytes and terminate. This is what snprintf and strlcpy do and it removes the classic strcpy overflow. Verified by compiling and running the solution with gcc.

---

More free practice: the [C Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
