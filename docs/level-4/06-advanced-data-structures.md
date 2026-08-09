# 06 · Advanced Data Structures

[Level 3](../level-3/02-trees-graphs.md) built a binary search tree that
works beautifully on random input and degenerates into a linked list on
sorted input — which is exactly the input real systems get, because real
data arrives in timestamp order, ID order, or alphabetical order.

This module covers the two structures that fix that in different ways: a
**self-balancing tree**, which guarantees the bound, and a **Bloom filter**,
which gives up exactness to buy a 10x reduction in memory. Both are chosen
because both make a trade you can measure.

## The problem, quantified

Insert 20,000 keys in sorted order into a plain BST and into an AVL tree:

```text
inserting 20000 keys in sorted order

structure      height ideal (log2 n)
AVL                15           14.3
plain BST       20000           14.3

lookup steps for 200 searches:
  AVL       : 2872
  plain BST : 1990200  (693x more work)
```

The plain BST's height equals its node count: every insert went right, so it
*is* a linked list with extra pointers. Its 693x penalty is not a constant
factor — it grows with n, so at 200,000 keys it would be roughly 7,000x.

The AVL tree's height of 15 against a theoretical minimum of 14.3 is the
guarantee in action: AVL keeps height within about 1.44·log₂(n), no matter
what order the keys arrive in.

## AVL trees: one invariant, four rotations

The rule is a single sentence: **for every node, the heights of its two
subtrees differ by at most 1.** Insert normally, then walk back up fixing
any node that violates it.

```c
// avl.c -- height bookkeeping
typedef struct Node {
    int key, height;
    struct Node *left, *right;
} Node;

static int h(Node *n)       { return n ? n->height : 0; }
static void fix(Node *n)    { n->height = 1 + max2(h(n->left), h(n->right)); }
static int balance(Node *n) { return n ? h(n->left) - h(n->right) : 0; }
```

Storing the height in the node is the whole trick. Computing it on demand
would be O(n) per check and defeat the purpose; keeping it current costs one
assignment per node on the way back up.

A rotation restructures three pointers and rewrites two heights:

```c
static Node *rot_right(Node *y) {
    Node *x = y->left;  y->left  = x->right; x->right = y;
    fix(y); fix(x); return x;          /* fix y BEFORE x: y is now the child */
}
static Node *rot_left(Node *x) {
    Node *y = x->right; x->right = y->left;  y->left  = x;
    fix(x); fix(y); return y;
}
```

The order of the two `fix` calls is load-bearing. After the rotation, `y` is
a child of `x`, so `x`'s height depends on `y`'s — recompute the child
first. Reversing these two lines produces a tree that looks correct, passes
casual testing, and silently stops balancing.

Insertion returns the (possibly new) subtree root, which is what lets the
caller re-link without a parent pointer:

```c
static Node *avl_insert(Node *n, int key) {
    if (!n) return make(key);
    if (key < n->key)      n->left  = avl_insert(n->left, key);
    else if (key > n->key) n->right = avl_insert(n->right, key);
    else return n;                       /* duplicate: no change */

    fix(n);
    int b = balance(n);
    if (b >  1 && key < n->left->key)  return rot_right(n);               /* LL */
    if (b < -1 && key > n->right->key) return rot_left(n);                /* RR */
    if (b >  1) { n->left  = rot_left(n->left);   return rot_right(n); }  /* LR */
    if (b < -1) { n->right = rot_right(n->right); return rot_left(n);  }  /* RL */
    return n;
}
```

Four cases, because a subtree can be too tall on the outside (one rotation)
or on the inside (two — first straighten, then rotate). `b > 1` means the
left side is too tall; comparing `key` against `n->left->key` distinguishes
outside from inside.

The `n->left = avl_insert(n->left, key)` pattern is the C idiom that makes
recursive tree code correct: the child always reassigns from the return
value, so a rotation at any depth propagates upward automatically and the
`if (!n) return make(key)` base case handles the empty tree with no special
case in the caller.

**Traps in this code specifically:**

- `h(NULL)` must return 0, not crash. Every height query goes through the
  null-checking helper; a direct `n->left->height` segfaults at every leaf.
- The recursion depth is bounded by the height, which is why the *balanced*
  version is also the one that will not blow the stack. A recursive insert
  into the degenerate BST above recurses 20,000 deep.
- Duplicate keys must be handled explicitly. Falling through to the
  balancing code after `return n` would compute a balance factor for an
  unchanged subtree — harmless here, but the same omission in `delete`
  corrupts the tree.

Red-black trees make a weaker guarantee (height ≤ 2·log₂ n) with fewer
rotations on insert, which is why `std::map` and most kernel structures use
them; AVL's tighter bound wins for lookup-heavy workloads. The invariant
and the rotation mechanics are the transferable part.

## Bloom filters: trading exactness for memory

Sometimes the question is not "what is the value?" but "is this worth
looking up at all?" — a cache checking whether a key could possibly be on
disk, a crawler checking whether it has seen a URL. A Bloom filter answers
that with a bit array and no stored keys at all.

Set `k` bits per item on insert; on query, if **any** of those bits is
clear, the item is definitely absent. If all are set, it is probably
present.

```c
// bloom.c -- constant space, no false negatives, some false positives
#define BITS (1u << 16)          /* 65536 bits = 8 KB */
#define K    3                   /* hash functions */

static uint8_t bits[BITS / 8];

static void set_bit(uint32_t i) { bits[i >> 3] |= (uint8_t)(1u << (i & 7)); }
static int  get_bit(uint32_t i) { return (bits[i >> 3] >> (i & 7)) & 1u; }

/* One FNV-1a walk, then derive K indices from it (Kirsch-Mitzenmacher). */
static void hashes(const char *s, uint32_t out[K]) {
    uint32_t h1 = 2166136261u, h2 = 5381u;
    for (const unsigned char *p = (const unsigned char *)s; *p; p++) {
        h1 = (h1 ^ *p) * 16777619u;
        h2 = h2 * 33u + *p;
    }
    for (int i = 0; i < K; i++)
        out[i] = (h1 + (uint32_t)i * h2) & (BITS - 1);
}

static int bloom_maybe(const char *s) {
    uint32_t idx[K]; hashes(s, idx);
    for (int i = 0; i < K; i++) if (!get_bit(idx[i])) return 0;  /* definitely absent */
    return 1;                                                    /* probably present */
}
```

5,000 members inserted, then 100,000 non-members probed:

```text
filter size    : 65536 bits (8192 bytes) for 5000 items
bits per item  : 13.1
false negatives: 0 (guaranteed to be 0)
false positives: 1077 of 100000 probes = 1.08%

a hash set of 5000 16-byte strings would need ~80000 bytes
```

Eight kilobytes instead of eighty, a 1.08% false-positive rate, and **zero**
false negatives — that last part is a structural guarantee, not a
measurement. A bit that was set is never cleared, so an inserted item can
never fail its own lookup.

The bit manipulation is worth reading closely. `i >> 3` is the byte index
(divide by 8) and `i & 7` is the bit within it (modulo 8), both valid only
because 8 is a power of two — the same trick as `& (BITS - 1)` for the array
index. See [Level 3's bit manipulation
module](../level-3/04-bit-manipulation.md).

Deriving `k` indices from two hashes rather than running `k` independent
hash functions is the Kirsch–Mitzenmacher optimisation: `h1 + i*h2` has
provably the same asymptotic false-positive rate at a third of the cost.

The design constraints, in the order they bite:

- **You cannot delete.** Clearing bits would break other members that share
  them. If you need deletion, you need a counting Bloom filter (a small
  counter per slot instead of a bit) at 4–8x the memory.
- **You cannot enumerate.** The filter stores no keys, only evidence.
- **Sizing is fixed up front.** False-positive rate ≈ (1 - e^(-kn/m))^k;
  around 10 bits per item with k=7 gives roughly 1%. Insert twice as many
  items as you planned and the rate degrades sharply — and silently.
- **Every hit needs verification.** A Bloom filter is a filter, not an
  answer. It saves the expensive lookup for the 99% it can rule out.

## Cheat sheet

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

## Exercise

Add `avl_delete` — the operation this module deliberately left out, and the
one that is genuinely harder than insert. Handle all three cases (leaf, one
child, two children with in-order-successor replacement) and rebalance on
the way back up, remembering that a single deletion can require rotations at
*every* level, unlike insertion which needs at most one.

Verify it with an invariant checker rather than by eye: write
`check(Node *)` that recursively asserts `abs(balance(n)) <= 1`, that the
stored `height` matches the recomputed one, and that an in-order walk is
strictly increasing. Then insert 10,000 random keys, delete them in random
order, and run `check` after **every** operation. Run the whole thing under
`-fsanitize=address` so a mistaken `free` of a node still linked into the
tree shows up immediately.

Then size a Bloom filter deliberately. Pick a target of 0.1% false positives
for 100,000 items, compute the required `m` and `k` from the formula, build
it, and measure whether you hit your target. Then insert 200,000 items into
that same filter and measure again — the number you get is the argument for
why a Bloom filter needs a documented capacity.
