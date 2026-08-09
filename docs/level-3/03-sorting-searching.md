# 03 · Sorting & Searching Algorithms

The standard library gives you `qsort` and `bsearch` for free, and in real
code you should almost always reach for those instead of hand-rolling your
own. This module builds a sort and a search by hand anyway, because
understanding *why* they're fast (and where they fall apart) is what tells
you when the library versions are the right tool and when they aren't —
and quicksort's partition step in particular shows up again, in a different
disguise, in [Module 10](10-project-kv-store.md)'s hash table.

## Quicksort: divide by partitioning

Quicksort picks a **pivot** value, rearranges the array so everything
smaller than the pivot ends up to its left and everything larger ends up to
its right (the pivot lands in its final sorted position in the process),
then recursively sorts the two sides. This implementation uses the Lomuto
partition scheme, which always picks the last element as pivot:

```c
// quicksort.c
#include <stdio.h>

void swap(int *a, int *b) {
    int tmp = *a;
    *a = *b;
    *b = tmp;
}

int partition(int arr[], int low, int high) {
    int pivot = arr[high];   // last element as pivot
    int i = low - 1;         // boundary of "smaller than pivot" region

    for (int j = low; j < high; j++) {
        if (arr[j] < pivot) {
            i++;
            swap(&arr[i], &arr[j]);
        }
    }
    swap(&arr[i + 1], &arr[high]);
    return i + 1;             // pivot's final index
}

void quicksort(int arr[], int low, int high) {
    if (low < high) {
        int pivot_index = partition(arr, low, high);
        quicksort(arr, low, pivot_index - 1);
        quicksort(arr, pivot_index + 1, high);
    }
}

void print_array(const int arr[], int n) {
    for (int i = 0; i < n; i++) {
        printf("%d ", arr[i]);
    }
    printf("\n");
}

int main(void) {
    int data[] = {9, 3, 7, 1, 8, 2, 5, 4, 6};
    int n = sizeof(data) / sizeof(data[0]);

    printf("before: ");
    print_array(data, n);

    quicksort(data, 0, n - 1);

    printf("after:  ");
    print_array(data, n);
    return 0;
}
```

```bash
gcc -Wall -Wextra -g -O0 -fsanitize=address,undefined -o quicksort quicksort.c
./quicksort
```

```text
before: 9 3 7 1 8 2 5 4 6
after:  1 2 3 4 5 6 7 8 9
```

Trace the first partition call (`partition(arr, 0, 8)`, pivot = `6`, the
last element): `j` scans `9, 3, 7, 1, 8, 2, 5, 4`, and every time it finds a
value less than `6` it bumps `i` and swaps that value into place. By the
time `j` reaches the end, everything less than `6` has been pushed to
indices `0..i`, and swapping `arr[i+1]` with the pivot drops `6` exactly
between the "less than" and "greater than" regions. That's the entire
algorithm — sorting is just repeating this split recursively on smaller and
smaller slices.

### The trap: worst-case quicksort is O(n²)

Average case, quicksort is O(n log n) — competitive with any general-purpose
sort. But Lomuto's "always pick the last element" pivot choice has a
specific weakness: feed it an **already-sorted** array, and every partition
call picks the *largest* remaining element as pivot, splitting the array
into a region of size `n-1` and a region of size `0`. That turns the
recursion into `n` nested calls instead of `log n`, and the whole sort
degrades to O(n²) — same asymptotic cost as bubble sort. Production
quicksort implementations dodge this by picking a random or median-of-three
pivot instead of always using the last element; the standard library's
`qsort` handles this (and the choice of algorithm entirely) for you, which
is the practical reason to prefer it once you understand what it's doing.

## Binary search: halving a sorted range

Binary search only works on already-sorted data, but in exchange it finds
any value in O(log n) comparisons instead of scanning linearly:

```c
// (append to quicksort.c)
int binary_search(const int arr[], int n, int target) {
    int low = 0, high = n - 1;
    while (low <= high) {
        int mid = low + (high - low) / 2;   // avoids overflow vs (low+high)/2
        if (arr[mid] == target) {
            return mid;
        } else if (arr[mid] < target) {
            low = mid + 1;
        } else {
            high = mid - 1;
        }
    }
    return -1;
}
```

```c
int main(void) {
    /* ... quicksort as above ... */
    int target = 7;
    int idx = binary_search(data, n, target);
    printf("binary_search(%d) = index %d\n", target, idx);

    target = 42;
    idx = binary_search(data, n, target);
    printf("binary_search(%d) = index %d\n", target, idx);
    return 0;
}
```

```text
binary_search(7) = index 6
binary_search(42) = index -1
```

`mid = low + (high - low) / 2` looks more roundabout than the textbook
`(low + high) / 2`, but it isn't cosmetic: if `low` and `high` are both
close to `INT_MAX`, `low + high` can overflow a signed `int`, which is
undefined behavior in C. `low + (high - low) / 2` computes the same
midpoint without ever adding two large values together. It rarely matters
for a 9-element array, but it's the kind of habit worth having before you're
searching a multi-gigabyte index where it does.

### Using the standard library instead

Once you understand the mechanics above, prefer the standard versions —
they're well-tested and handle any type via a comparator:

```c
#include <stdlib.h>

int compare_ints(const void *a, const void *b) {
    int ia = *(const int *)a;
    int ib = *(const int *)b;
    return (ia > ib) - (ia < ib);   // -1, 0, or 1 without risking overflow
}

qsort(data, n, sizeof(int), compare_ints);
int target = 7;
int *found = bsearch(&target, data, n, sizeof(int), compare_ints);
```

`qsort` and `bsearch` both take a `size_t` element size and a comparator
function pointer, which is how they sort/search arrays of *any* type — the
tradeoff for that generality is the `void *` casts and the indirect function
call on every comparison, which is measurably slower than the specialized
`int`-only version above for hot loops.

## Complexity cheat sheet

| Algorithm | Best | Average | Worst | Notes |
|-----------|------|---------|-------|-------|
| Quicksort | O(n log n) | O(n log n) | O(n²) | In-place; worst case on already-sorted input with a naive pivot |
| Merge sort | O(n log n) | O(n log n) | O(n log n) | Stable, but needs O(n) extra space |
| Bubble sort | O(n) | O(n²) | O(n²) | Simple, never used in practice above tiny n |
| Linear search | O(1) | O(n) | O(n) | Works on unsorted data |
| Binary search | O(1) | O(log n) | O(log n) | Requires sorted data |

## Exercise

Modify `partition` to pick the pivot as the **middle** element of the range
instead of the last one (swap `arr[mid]` and `arr[high]` at the top of
`partition`, before the existing logic runs unchanged). Recompile with
`-fsanitize=address,undefined` and confirm sorting still produces the same
correct output on the sample array. Then construct an already-sorted 20
element array (`1, 2, 3, ..., 20`) and add a counter that increments on
every `partition` call; compare the call count between the last-element
pivot and the middle-element pivot to see the worst-case behavior
concretely instead of just taking it on faith.
