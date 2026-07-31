# 10 · Project — Linked-list Inventory System

This project pulls together everything from Level 2: pointers, dynamic memory,
structs, binary file I/O, error handling, modular design, and a real Makefile.
Unlike Level 1's array-based contact book, this inventory is a **singly linked
list** — so adding an item never needs to resize or copy anything, and you get
hands-on practice with the pointer-chasing that Level 3 topics build on.

## What you'll build

A command-line inventory tracker that:

- Adds items (SKU, name, quantity, unit price) as linked-list nodes
- Lists all items and computes total inventory value
- Adjusts stock for a SKU (receiving or shipping) without ever going negative
- Persists everything to a **binary** file between runs, using the techniques
  from [Module 5](05-binary-file-io.md) — the list is flattened to an array on
  save and rebuilt into a list on load
- Builds with a real Makefile with `-MMD -MP` dependency tracking

## Project layout

```text
inventory/
    Makefile
    include/
        inventory.h
    src/
        inventory.c
        main.c
```

## include/inventory.h — the module's public contract

Following the opaque-type pattern from [Module 7](07-modular-programming.md):
`main.c` never touches a `Node` directly, only through these functions.

```c
// inventory.h
#ifndef INVENTORY_H
#define INVENTORY_H

#define SKU_LEN  16
#define NAME_LEN 48

typedef struct {
    char sku[SKU_LEN];
    char name[NAME_LEN];
    int quantity;
    double unit_price;
} Item;

// The node type is declared here (not hidden) because inventory_first /
// inventory_next let main.c walk the list to print it -- but its fields
// are still only ever touched through the accessors below.
typedef struct Node {
    Item item;
    struct Node *next;
} Node;

typedef struct {
    Node *head;
    size_t count;
} Inventory;

void inventory_init(Inventory *inv);
void inventory_destroy(Inventory *inv);   // frees every node

// Returns 1 on success, 0 on failure (out of memory, or duplicate SKU).
int inventory_add(Inventory *inv, const char *sku, const char *name,
                   int quantity, double unit_price);

// Returns a pointer into the list, or NULL if the SKU isn't found.
// Callers must not free this pointer -- the inventory owns it.
Item *inventory_find(Inventory *inv, const char *sku);

// delta may be negative (shipping stock out). Returns 0 if it would
// take quantity below zero, and leaves the item unchanged.
int inventory_adjust_stock(Inventory *inv, const char *sku, int delta);

// Removes the node for sku. Returns 0 if not found.
int inventory_remove(Inventory *inv, const char *sku);

double inventory_total_value(const Inventory *inv);

// List traversal for callers that just want to print everything.
const Node *inventory_first(const Inventory *inv);
const Node *inventory_next(const Node *node);

// Binary persistence -- flattens to an array of Item on save,
// rebuilds the list on load. See Module 5 for the file format.
int inventory_save(const Inventory *inv, const char *path);
int inventory_load(Inventory *inv, const char *path);

#endif
```

## src/inventory.c — implementation

```c
// inventory.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "inventory.h"

void inventory_init(Inventory *inv) {
    inv->head = NULL;
    inv->count = 0;
}

void inventory_destroy(Inventory *inv) {
    Node *cur = inv->head;
    while (cur != NULL) {
        Node *next = cur->next;   // save next BEFORE freeing cur
        free(cur);
        cur = next;
    }
    inv->head = NULL;
    inv->count = 0;
}

Item *inventory_find(Inventory *inv, const char *sku) {
    for (Node *cur = inv->head; cur != NULL; cur = cur->next) {
        if (strcmp(cur->item.sku, sku) == 0) {
            return &cur->item;
        }
    }
    return NULL;
}

int inventory_add(Inventory *inv, const char *sku, const char *name,
                   int quantity, double unit_price) {
    if (inventory_find(inv, sku) != NULL) {
        return 0;   // duplicate SKU -- use inventory_adjust_stock instead
    }

    Node *node = malloc(sizeof(Node));
    if (node == NULL) {
        return 0;
    }

    strncpy(node->item.sku, sku, SKU_LEN - 1);
    node->item.sku[SKU_LEN - 1] = '\0';
    strncpy(node->item.name, name, NAME_LEN - 1);
    node->item.name[NAME_LEN - 1] = '\0';
    node->item.quantity = quantity;
    node->item.unit_price = unit_price;

    // Insert at the head -- O(1), and order doesn't matter for this use case.
    node->next = inv->head;
    inv->head = node;
    inv->count++;
    return 1;
}

int inventory_adjust_stock(Inventory *inv, const char *sku, int delta) {
    Item *item = inventory_find(inv, sku);
    if (item == NULL) {
        return 0;
    }
    if (item->quantity + delta < 0) {
        return 0;   // would go negative -- refuse rather than corrupt state
    }
    item->quantity += delta;
    return 1;
}

int inventory_remove(Inventory *inv, const char *sku) {
    Node *cur = inv->head;
    Node *prev = NULL;

    while (cur != NULL) {
        if (strcmp(cur->item.sku, sku) == 0) {
            if (prev == NULL) {
                inv->head = cur->next;   // removing the head node
            } else {
                prev->next = cur->next; // splice cur out of the chain
            }
            free(cur);
            inv->count--;
            return 1;
        }
        prev = cur;
        cur = cur->next;
    }
    return 0;   // not found
}

double inventory_total_value(const Inventory *inv) {
    double total = 0.0;
    for (Node *cur = inv->head; cur != NULL; cur = cur->next) {
        total += cur->item.quantity * cur->item.unit_price;
    }
    return total;
}

const Node *inventory_first(const Inventory *inv) {
    return inv->head;
}

const Node *inventory_next(const Node *node) {
    return node->next;
}

int inventory_save(const Inventory *inv, const char *path) {
    FILE *f = fopen(path, "wb");
    if (f == NULL) {
        return 0;
    }

    // Header first: how many records follow, so inventory_load doesn't
    // have to guess -- the same self-describing format Module 5 covers.
    if (fwrite(&inv->count, sizeof(inv->count), 1, f) != 1) {
        fclose(f);
        return 0;
    }

    // Flatten the list to a contiguous array purely for the write --
    // the file format is an array; the in-memory structure is a list.
    Item *buffer = malloc(inv->count * sizeof(Item));
    if (inv->count > 0 && buffer == NULL) {
        fclose(f);
        return 0;
    }

    size_t i = 0;
    for (Node *cur = inv->head; cur != NULL; cur = cur->next) {
        buffer[i++] = cur->item;
    }

    size_t written = fwrite(buffer, sizeof(Item), inv->count, f);
    free(buffer);
    fclose(f);
    return written == inv->count;
}

int inventory_load(Inventory *inv, const char *path) {
    inventory_init(inv);

    FILE *f = fopen(path, "rb");
    if (f == NULL) {
        return 1;   // no file yet -- an empty inventory is a valid result
    }

    size_t count;
    if (fread(&count, sizeof(count), 1, f) != 1) {
        fclose(f);
        return 1;   // empty or corrupt header -- start empty rather than fail
    }

    Item *buffer = malloc(count * sizeof(Item));
    if (count > 0 && buffer == NULL) {
        fclose(f);
        return 0;
    }

    size_t read_count = fread(buffer, sizeof(Item), count, f);
    fclose(f);

    if (read_count != count) {
        free(buffer);
        return 1;   // truncated file -- don't trust it, start empty
    }

    // Rebuild the list from the flat array, preserving on-disk order
    // (append at the tail this time, since order came from the file).
    Node *tail = NULL;
    for (size_t i = 0; i < count; i++) {
        Node *node = malloc(sizeof(Node));
        if (node == NULL) {
            free(buffer);
            inventory_destroy(inv);   // clean up partial list before failing
            return 0;
        }
        node->item = buffer[i];
        node->next = NULL;

        if (tail == NULL) {
            inv->head = node;
        } else {
            tail->next = node;
        }
        tail = node;
        inv->count++;
    }

    free(buffer);
    return 1;
}
```

## src/main.c — CLI

```c
// main.c
#include <stdio.h>
#include <string.h>
#include "inventory.h"

#define DB_PATH "inventory.dat"

static void print_menu(void) {
    printf("\n1. Add item  2. List  3. Adjust stock  4. Save & Quit\n> ");
}

int main(void) {
    Inventory inv;
    if (!inventory_load(&inv, DB_PATH)) {
        fprintf(stderr, "Could not initialize inventory.\n");
        return 1;
    }

    int choice;
    char sku[SKU_LEN], name[NAME_LEN];

    do {
        print_menu();
        if (scanf("%d", &choice) != 1) {
            break;
        }
        getchar();   // consume the leftover newline

        switch (choice) {
            case 1: {
                int qty;
                double price;
                printf("SKU: ");
                fgets(sku, SKU_LEN, stdin);
                sku[strcspn(sku, "\n")] = '\0';
                printf("Name: ");
                fgets(name, NAME_LEN, stdin);
                name[strcspn(name, "\n")] = '\0';
                printf("Quantity: ");
                scanf("%d", &qty);
                printf("Unit price: ");
                scanf("%lf", &price);
                getchar();

                if (inventory_add(&inv, sku, name, qty, price)) {
                    printf("Added.\n");
                } else {
                    printf("Failed -- duplicate SKU or out of memory.\n");
                }
                break;
            }
            case 2:
                for (const Node *cur = inventory_first(&inv); cur != NULL;
                     cur = inventory_next(cur)) {
                    printf("%-8s %-24s qty=%-5d $%.2f\n",
                           cur->item.sku, cur->item.name,
                           cur->item.quantity, cur->item.unit_price);
                }
                printf("Total value: $%.2f\n", inventory_total_value(&inv));
                break;

            case 3: {
                int delta;
                printf("SKU: ");
                fgets(sku, SKU_LEN, stdin);
                sku[strcspn(sku, "\n")] = '\0';
                printf("Adjustment (negative to ship out): ");
                scanf("%d", &delta);
                getchar();

                if (inventory_adjust_stock(&inv, sku, delta)) {
                    printf("Updated.\n");
                } else {
                    printf("Failed -- unknown SKU or insufficient stock.\n");
                }
                break;
            }

            case 4:
                if (inventory_save(&inv, DB_PATH)) {
                    printf("Saved. Goodbye!\n");
                } else {
                    printf("Warning: save failed.\n");
                }
                break;

            default:
                printf("Unknown option.\n");
        }
    } while (choice != 4);

    inventory_destroy(&inv);
    return 0;
}
```

## Makefile

Following [Module 8](08-makefiles-build-systems.md)'s pattern rules and
automatic dependency generation:

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
	rm -rf $(BUILD_DIR) inventory.dat

-include $(DEPS)
```

## Running it

```bash
make
make run
```

```text
1. Add item  2. List  3. Adjust stock  4. Save & Quit
> 1
SKU: WGT-100
Name: Widget
Quantity: 50
Unit price: 4.25
Added.

1. Add item  2. List  3. Adjust stock  4. Save & Quit
> 2
WGT-100  Widget                   qty=50    $4.25
Total value: $212.50

1. Add item  2. List  3. Adjust stock  4. Save & Quit
> 3
SKU: WGT-100
Adjustment (negative to ship out): -60
Failed -- unknown SKU or insufficient stock.

1. Add item  2. List  3. Adjust stock  4. Save & Quit
> 4
Saved. Goodbye!
```

The rejected `-60` adjustment against 50 units in stock is
`inventory_adjust_stock` doing exactly its job — refusing to let quantity go
negative rather than silently corrupting the record.

Notice that inserting at the **head** means the newest item prints first on
`list` — a direct, visible consequence of the O(1) insert design. If insertion
order needs to be preserved for display, that's what `inventory_load`'s
tail-append does when rebuilding from a file, and is a natural extension for
`inventory_add` too.

## Verifying no leaks

This project allocates a `Node` per item and frees them individually in
`inventory_destroy` and `inventory_remove` — exactly the kind of code where a
missed `free` or a use-after-free hides easily. Run it under the tools from
[Module 9](09-debugging-tools.md):

```bash
gcc -Wall -Wextra -g -O0 -Iinclude -fsanitize=address,undefined \
    -o inventory_debug src/inventory.c src/main.c
./inventory_debug
```

Add a few items, remove one, adjust stock, save and quit — a clean exit with
no sanitizer output means every node allocated was freed exactly once.

## Stretch goals

- Add a `4b. Remove item` menu option calling `inventory_remove`, and confirm
  with the sanitizer that removing the **head**, a **middle**, and the
  **only** node all work without leaking or double-freeing.
- Add a low-stock report: walk the list and print every item with `quantity`
  below a caller-supplied threshold.
- Reverse the list in place (swap `next` pointers without allocating any new
  nodes) and add a menu option to toggle display order.
- Run the full program under `valgrind --leak-check=full` and confirm
  `All heap blocks were freed -- no leaks are possible`.

Completing this project means you're ready for **Level 3 · Advanced**.
