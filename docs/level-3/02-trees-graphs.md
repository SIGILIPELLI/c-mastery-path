# 02 · Trees & Basic Graph Representations

[Module 01](01-data-structures.md)'s linked list chains nodes one after
another. A **tree** relaxes that to let each node point at *multiple*
children, and a **graph** relaxes it further so connections don't even have
to flow in one direction. Both are still just structs and pointers — the
only thing that changes is the shape of the relationships.

## Binary search trees

A binary search tree (BST) keeps every value in its left subtree smaller
than the node, and every value in its right subtree larger. That ordering
is what makes search fast: at each node you eliminate half the remaining
tree, the same way you'd search a sorted array by halves.

```c
// bst.c
#include <stdio.h>
#include <stdlib.h>

typedef struct TreeNode {
    int value;
    struct TreeNode *left;
    struct TreeNode *right;
} TreeNode;

TreeNode *tree_insert(TreeNode *root, int value) {
    if (root == NULL) {
        TreeNode *n = malloc(sizeof(TreeNode));
        if (!n) { fprintf(stderr, "out of memory\n"); exit(1); }
        n->value = value;
        n->left = n->right = NULL;
        return n;
    }
    if (value < root->value) {
        root->left = tree_insert(root->left, value);
    } else if (value > root->value) {
        root->right = tree_insert(root->right, value);
    }
    // equal values are ignored -- no duplicates in this tree
    return root;
}

TreeNode *tree_find(TreeNode *root, int value) {
    if (root == NULL || root->value == value) {
        return root;
    }
    if (value < root->value) {
        return tree_find(root->left, value);
    }
    return tree_find(root->right, value);
}

void tree_print_inorder(const TreeNode *root) {
    if (root == NULL) return;
    tree_print_inorder(root->left);
    printf("%d ", root->value);
    tree_print_inorder(root->right);
}

void tree_free(TreeNode *root) {
    if (root == NULL) return;
    tree_free(root->left);
    tree_free(root->right);
    free(root);
}

int main(void) {
    TreeNode *root = NULL;
    int values[] = {50, 30, 70, 20, 40, 60, 80};
    for (size_t i = 0; i < sizeof(values) / sizeof(values[0]); i++) {
        root = tree_insert(root, values[i]);
    }

    printf("in-order: ");
    tree_print_inorder(root);
    printf("\n");

    int target = 40;
    TreeNode *found = tree_find(root, target);
    printf("find(%d): %s\n", target, found ? "found" : "not found");

    target = 99;
    found = tree_find(root, target);
    printf("find(%d): %s\n", target, found ? "found" : "not found");

    tree_free(root);
    return 0;
}
```

```bash
gcc -Wall -Wextra -g -O0 -fsanitize=address,undefined -o bst bst.c
./bst
```

```text
in-order: 20 30 40 50 60 70 80
find(40): found
find(99): not found
```

Two things worth noticing:

- **`tree_insert` returns a `TreeNode *` that callers must reassign**, exactly
  like `push_front` in Module 01. `root->left = tree_insert(root->left, value)`
  is what actually attaches a freshly allocated node into the tree — miss the
  assignment and the new node is allocated, then immediately leaked.
- **In-order traversal (left, node, right) visits a BST's values in sorted
  order** — that's not a coincidence, it falls directly out of the ordering
  invariant. It's also why the pattern (`recurse left`, `visit`, `recurse
  right`) is worth memorizing; swapping the visit to before or after the
  recursive calls gives you pre-order and post-order instead, useful for
  different jobs (pre-order to copy a tree, post-order to free one bottom-up,
  which is exactly what `tree_free` does).

### The trap: an unbalanced tree degrades to a linked list

`tree_insert`'s O(log n) search only holds if the tree stays roughly
balanced. Insert already-sorted data (`1, 2, 3, 4, 5...`) and every new node
becomes the right child of the previous one — you get a straight chain, and
search degrades to O(n). Real-world BST-based containers (like C++'s
`std::map` or Java's `TreeMap`) use self-balancing variants (red-black
trees, AVL trees) specifically to guarantee this can't happen; a plain BST
like this one doesn't protect you from it.

## Graphs: adjacency list representation

A graph is a set of vertices plus a set of edges connecting them, and edges
don't have to form a hierarchy — cycles are allowed, and a vertex can
connect to any number of others. The two standard ways to represent one in
C are an **adjacency matrix** (an N×N grid of 0/1) and an **adjacency
list** (each vertex keeps a linked list of its neighbors). Adjacency lists
win for sparse graphs — most real graphs (social networks, road maps,
dependency graphs) — because memory scales with the number of edges, not
the square of the vertex count.

```c
// graph_bfs.c
#include <stdio.h>
#include <stdlib.h>

#define MAX_VERTICES 6

typedef struct EdgeNode {
    int dest;
    struct EdgeNode *next;
} EdgeNode;

typedef struct {
    EdgeNode *adj[MAX_VERTICES];   // one linked list of neighbors per vertex
    int num_vertices;
} Graph;

void graph_init(Graph *g, int num_vertices) {
    g->num_vertices = num_vertices;
    for (int i = 0; i < num_vertices; i++) {
        g->adj[i] = NULL;
    }
}

void graph_add_edge(Graph *g, int src, int dest) {
    // undirected: add dest to src's list, and src to dest's list
    EdgeNode *n1 = malloc(sizeof(EdgeNode));
    n1->dest = dest;
    n1->next = g->adj[src];
    g->adj[src] = n1;

    EdgeNode *n2 = malloc(sizeof(EdgeNode));
    n2->dest = src;
    n2->next = g->adj[dest];
    g->adj[dest] = n2;
}

void graph_bfs(const Graph *g, int start) {
    int visited[MAX_VERTICES] = {0};
    int queue[MAX_VERTICES];
    int head = 0, tail = 0;

    visited[start] = 1;
    queue[tail++] = start;

    printf("BFS from %d: ", start);
    while (head < tail) {
        int current = queue[head++];
        printf("%d ", current);

        for (EdgeNode *e = g->adj[current]; e != NULL; e = e->next) {
            if (!visited[e->dest]) {
                visited[e->dest] = 1;
                queue[tail++] = e->dest;
            }
        }
    }
    printf("\n");
}

void graph_free(Graph *g) {
    for (int i = 0; i < g->num_vertices; i++) {
        EdgeNode *cur = g->adj[i];
        while (cur != NULL) {
            EdgeNode *next = cur->next;
            free(cur);
            cur = next;
        }
    }
}

int main(void) {
    Graph g;
    graph_init(&g, 6);

    //    0 -- 1 -- 3
    //    |    |
    //    2    4 -- 5
    graph_add_edge(&g, 0, 1);
    graph_add_edge(&g, 0, 2);
    graph_add_edge(&g, 1, 3);
    graph_add_edge(&g, 1, 4);
    graph_add_edge(&g, 4, 5);

    graph_bfs(&g, 0);

    graph_free(&g);
    return 0;
}
```

```text
BFS from 0: 0 2 1 4 3 5
```

`graph_bfs` uses a plain fixed-size array as its frontier queue, not the
circular buffer from Module 01 — a simple `head`/`tail` pair is enough here
because each vertex is enqueued at most once (guarded by `visited`), so
`tail` can never overrun `MAX_VERTICES` slots. The visit order (`0 2 1 4 3 5`) depends on the order neighbors
were linked into each adjacency list — `graph_add_edge` prepends, so the
*last* edge added to a vertex is the *first* one BFS explores from it.

Depth-first search (DFS) uses the identical structure with one change:
replace the FIFO `queue` with the stack from Module 01 (or a recursive call,
which uses the *call* stack instead of an explicit one) — same visited-set
bookkeeping, different order of exploration.

## Tree/graph terms cheat sheet

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

## Exercise

Add a `tree_height(const TreeNode *root)` function to `bst.c` that returns
`0` for an empty tree and `1 + max(height(left), height(right))` otherwise.
Run it against the tree built in `main` (`50, 30, 70, 20, 40, 60, 80`) and
confirm it prints `3`. Then insert values `1, 2, 3, 4, 5` into a *fresh*
empty tree (nothing else) and print the height again — it should come out
as `5`, demonstrating the unbalanced-degrades-to-a-list problem described
above.
