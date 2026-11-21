---
title: "Final Quiz"
description: "Test what you have learned across all four levels of C Mastery Path with a 20-question final quiz."
---

# Final Quiz

20 questions drawn from all four levels of this course. You need **70% (14 out of 20)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this print?\n```c\n#include <stdio.h>\nint main(void){ int a=7,b=2; printf(\"%d %.1f %d\", a/b, (double)a/b, a%b); return 0; }\n```\n",
    "options": [
      "3.5 3.5 1",
      "4 3.5 1",
      "3 3.5 1",
      "3 3 1"
    ],
    "correctIndex": 2,
    "explanation": "Integer division truncates; cast one operand to double for a fractional result. Verified by compiling and running."
  },
  {
    "question": "What does this print?\n```c\n#include <stdio.h>\nint main(void){ int n=2; switch(n){ case 1: printf(\"a\"); case 2: printf(\"b\"); case 3: printf(\"c\"); break; default: printf(\"d\"); } return 0; }\n```\n",
    "options": [
      "bcd",
      "c",
      "bc",
      "b"
    ],
    "correctIndex": 2,
    "explanation": "Missing break falls through case 3 and stops at break. Verified by compiling and running."
  },
  {
    "question": "What does this print?\n```c\n#include <stdio.h>\nvoid f(int *p, int v){ *p = v; v = 99; }\nint main(void){ int x=1,y=2; f(&x,y); printf(\"%d %d\", x, y); return 0; }\n```\n",
    "options": [
      "1 2",
      "2 2",
      "2 99",
      "99 2"
    ],
    "correctIndex": 1,
    "explanation": "The pointer lets f change x; v is a copy, so y is untouched. Verified by compiling and running."
  },
  {
    "question": "What does this print?\n```c\n#include <stdio.h>\n#define DOUBLE(x) x*2\nint main(void){ printf(\"%d\", DOUBLE(3+4)); return 0; }\n```\n",
    "options": [
      "14",
      "7",
      "24",
      "11"
    ],
    "correctIndex": 3,
    "explanation": "The macro expands to 3+4*2, which is 11, not 14: wrap macro parameters in parentheses. Verified by compiling and running."
  },
  {
    "question": "What does this print?\n```c\n#include <stdio.h>\nint main(void){ int a[]={4,8,15,16}; int *p=a+1; printf(\"%d %d %d\", *p, p[2], (int)(&a[3]-p)); return 0; }\n```\n",
    "options": [
      "8 16 2",
      "4 15 2",
      "8 15 3",
      "8 16 8"
    ],
    "correctIndex": 0,
    "explanation": "p points at a[1]; p[2] is a[3]; pointer difference counts elements. Verified by compiling and running."
  },
  {
    "question": "What does this print (typical 64-bit)?\n```c\n#include <stdio.h>\ntypedef struct { char a; double b; char c; } S;\nint main(void){ printf(\"%zu\", sizeof(S)); return 0; }\n```\n",
    "options": [
      "24",
      "16",
      "10",
      "12"
    ],
    "correctIndex": 0,
    "explanation": "Padding aligns the double to 8 bytes: 1+7 pad, 8, 1+7 pad. Verified by compiling and running."
  },
  {
    "question": "What does this print?\n```c\n#include <stdio.h>\n#include <string.h>\nint main(void){ char s[16]=\"hello\"; printf(\"%zu %zu %c\", strlen(s), sizeof s, s[strlen(s)]=='\\0'?'Y':'N'); return 0; }\n```\n",
    "options": [
      "5 16 N",
      "5 16 Y",
      "5 5 Y",
      "6 16 Y"
    ],
    "correctIndex": 1,
    "explanation": "strlen stops at the terminator; sizeof is the whole buffer; the byte after the last char is '\\0'. Verified by compiling and running."
  },
  {
    "question": "What does this print?\n```c\n#include <stdio.h>\nint main(void){ unsigned x=0x2C; printf(\"%u %u %u\", x&0x0F, x|0x01, x>>2); return 0; }\n```\n",
    "options": [
      "12 44 11",
      "12 45 11",
      "44 45 11",
      "12 45 22"
    ],
    "correctIndex": 1,
    "explanation": "0x2C = 44: & 0x0F gives 12, | 1 gives 45, >> 2 gives 11. Verified by compiling and running."
  },
  {
    "question": "Why does `p = realloc(p, newsize)` risk a leak?",
    "options": [
      "It zeroes memory",
      "realloc always copies twice",
      "It frees the pointer twice",
      "On failure realloc returns NULL, overwriting p and losing the original block; assign to a temporary first"
    ],
    "correctIndex": 3,
    "explanation": "Check the temp for NULL, then assign it to p."
  },
  {
    "question": "A function returns a pointer to a local array. What's wrong?",
    "options": [
      "The array lives on the stack and is gone when the function returns: a dangling pointer",
      "It's a syntax error",
      "Nothing",
      "It leaks"
    ],
    "correctIndex": 0,
    "explanation": "Return heap memory, a static buffer, or write into a caller-provided buffer."
  },
  {
    "question": "You free a node then read its `next`. What happened?",
    "options": [
      "A stack overflow",
      "A double free",
      "Use-after-free: save next before freeing",
      "Nothing wrong"
    ],
    "correctIndex": 2,
    "explanation": "Undefined behavior; ASan reports it."
  },
  {
    "question": "`make` rebuilds everything after you edit only a header. Which fix is correct?",
    "options": [
      "Use := everywhere",
      "Use -O3",
      "Delete the headers",
      "Generate dependency files with -MMD -MP and include them so objects depend on the headers they use (and the opposite problem, stale builds, is also avoided)"
    ],
    "correctIndex": 3,
    "explanation": "Edited headers must invalidate dependent objects."
  },
  {
    "question": "A server reads a request with a single recv() into a 64-byte buffer. What can go wrong?",
    "options": [
      "Messages are always whole",
      "Nothing",
      "TCP is a byte stream: the read may return a partial message, and unbounded copying could overflow the buffer; loop and bound it",
      "recv blocks forever"
    ],
    "correctIndex": 2,
    "explanation": "Check the return value and size every copy."
  },
  {
    "question": "Two threads increment a shared counter 1,000,000 times each with no synchronization. What do you expect?",
    "options": [
      "Exactly 1,000,000",
      "A total below 2,000,000 because increments are read-modify-write and updates are lost; use an atomic or mutex",
      "Exactly 2,000,000",
      "A compile error"
    ],
    "correctIndex": 1,
    "explanation": "A data race is undefined behavior."
  },
  {
    "question": "Which is the safest way to build a path from user input into a 64-byte buffer?",
    "options": [
      "strcpy then strcat",
      "sprintf without checks",
      "snprintf(buf, sizeof buf, \"%s/%s\", dir, name) and check that the result is less than sizeof buf",
      "gets"
    ],
    "correctIndex": 2,
    "explanation": "snprintf never writes past the size and reports truncation."
  },
  {
    "question": "Why compile and test with -fsanitize=address,undefined?",
    "options": [
      "They enable threads",
      "They remove the need for tests",
      "They turn silent memory errors and undefined behavior into immediate, located reports",
      "They speed up the program"
    ],
    "correctIndex": 2,
    "explanation": "Use them in CI along with warnings -Wall -Wextra."
  },
  {
    "question": "Which structure gives guaranteed O(log n) search with inserts and deletes?",
    "options": [
      "A linked list",
      "A self-balancing tree such as an AVL tree",
      "An unsorted array",
      "A plain BST with sorted input"
    ],
    "correctIndex": 1,
    "explanation": "Rotations keep the height logarithmic."
  },
  {
    "question": "You need many small short-lived allocations freed all at once (e.g. per request). Which allocator fits?",
    "options": [
      "malloc/free for each",
      "An arena: allocation is a pointer bump and reset frees everything",
      "The stack always",
      "A linked list"
    ],
    "correctIndex": 1,
    "explanation": "Lifetimes are grouped."
  },
  {
    "question": "What should an exit code of a test runner convey to CI?",
    "options": [
      "Non-zero means at least one test failed",
      "Always zero",
      "Zero means failed",
      "The number of lines"
    ],
    "correctIndex": 0,
    "explanation": "CI gates on the exit status."
  },
  {
    "question": "What is the safest design for a C function that can fail and returns a value?",
    "options": [
      "Call exit",
      "Return -1 for all failures",
      "Return a status code and deliver the value through an out-parameter, documenting ownership of any allocation",
      "Ignore errors"
    ],
    "correctIndex": 2,
    "explanation": "Sentinels can collide with valid values."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Reverse a linked list in place

Implement reverse(): re-point each node's next pointer so the list is reversed without allocating new nodes, and return the new head. The program pushes 1..4 at the head (so it prints 4 3 2 1) and then prints the reversed list.

**Starter code**

```python
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

**Expected output**

```text
4 3 2 1
1 2 3 4
```

<details><summary>Show solution and explanation</summary>

<pre><code>#include &lt;stdio.h&gt;
#include &lt;stdlib.h&gt;
typedef struct Node { int v; struct Node *next; } Node;

Node *push(Node *head, int v) {
    Node *n = malloc(sizeof *n);
    n-&gt;v = v; n-&gt;next = head;
    return n;
}
Node *reverse(Node *head) {
    Node *prev = NULL;
    while (head) {
        Node *next = head-&gt;next;
        head-&gt;next = prev;
        prev = head;
        head = next;
    }
    return prev;
}
void print(Node *h) { for (; h; h = h-&gt;next) printf(&quot;%d%c&quot;, h-&gt;v, h-&gt;next ? &#x27; &#x27; : &#x27;\n&#x27;); }
int main(void) {
    Node *h = NULL;
    for (int i = 1; i &lt;= 4; i++) h = push(h, i);
    print(h);
    h = reverse(h);
    print(h);
    while (h) { Node *n = h-&gt;next; free(h); h = n; }
    return 0;
}
</code></pre>

<p>Walk the list keeping prev, save next before overwriting, and return prev as the new head. It&#x27;s O(n) time and O(1) extra space. Verified by compiling and running the solution with gcc.</p>
</details>

### A bounded string copy

Implement safe_copy(dst, cap, src): copy src into dst without overflowing, always null-terminate when cap > 0, and return 1 if everything fit or 0 if it was truncated. Print the result for a 6-byte and a 32-byte destination.

**Starter code**

```python
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

**Expected output**

```text
0 [Maste]
1 [Mastery]
```

<details><summary>Show solution and explanation</summary>

<pre><code>#include &lt;stdio.h&gt;
#include &lt;string.h&gt;

/* copy src into dst (capacity cap); return 1 if it all fit, 0 if truncated.
   dst is always null-terminated when cap &gt; 0. */
int safe_copy(char *dst, size_t cap, const char *src) {
    if (cap == 0) return 0;
    size_t n = strlen(src);
    if (n &gt;= cap) {
        memcpy(dst, src, cap - 1);
        dst[cap - 1] = &#x27;\0&#x27;;
        return 0;
    }
    memcpy(dst, src, n + 1);
    return 1;
}
int main(void) {
    char small[6], big[32];
    int a = safe_copy(small, sizeof small, &quot;Mastery&quot;);
    int b = safe_copy(big, sizeof big, &quot;Mastery&quot;);
    printf(&quot;%d [%s]\n%d [%s]\n&quot;, a, small, b, big);
    return 0;
}
</code></pre>

<p>Compare strlen(src) with cap: if it doesn&#x27;t fit, copy cap-1 bytes and terminate. This is what snprintf and strlcpy do and it removes the classic strcpy overflow. Verified by compiling and running the solution with gcc.</p>
</details>
