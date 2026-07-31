# 05 · Binary File I/O

[Level 1, Module 8](../level-1/08-file-io.md) wrote files with `fprintf` — the
number `1000000` became the seven characters `1000000`. That's *text* I/O:
human-readable, easy to debug, and lossy for floats. **Binary** I/O writes the
bytes of the value itself: those same 1,000,000 become four bytes, exactly as
they sit in memory, with no conversion and no rounding.

Binary files are smaller, faster to read and write, and support *random access*
— you can jump directly to record 500 without reading the first 499. The trade
is that you can no longer inspect the file with `cat`, and the format is tied to
your machine's representation unless you're careful.

## Text vs binary at a glance

| | Text (`fprintf`/`fscanf`) | Binary (`fwrite`/`fread`) |
|---|---------------------------|----------------------------|
| `int 1000000` on disk | 7 bytes: `1000000` | 4 bytes: `40 42 0F 00` |
| `double 0.1` | rounded to your `%f` precision | exact bit pattern |
| Human readable | yes | no |
| Parse cost | must convert every field | none — it's already a value |
| Random access | hard (variable-width lines) | easy (fixed-size records) |
| Portable across machines | yes | not automatically |

## Opening in binary mode

Add `b` to the mode string. On Linux and macOS it changes nothing; on Windows
it stops the C library translating `\n` into `\r\n`, which would corrupt your
data. **Always include it** — it costs nothing and makes the code portable.

| Mode | Meaning |
|------|---------|
| `"rb"` | read; fails if the file doesn't exist |
| `"wb"` | write; truncates an existing file to zero length |
| `"ab"` | append; writes always go to the end |
| `"rb+"` | read **and** write an existing file (for updating records in place) |
| `"wb+"` | read and write, truncating first |

## `fwrite` and `fread`

Both have the same shape:

```c
size_t fwrite(const void *ptr, size_t size, size_t count, FILE *stream);
size_t fread (      void *ptr, size_t size, size_t count, FILE *stream);
```

They move `size * count` bytes and return the number of **items** (not bytes)
successfully transferred. A short return means end-of-file or an error.

```c
#include <stdio.h>

int main(void) {
    int numbers[] = {10, 20, 30, 40, 50};
    size_t n = sizeof numbers / sizeof numbers[0];

    FILE *f = fopen("numbers.bin", "wb");
    if (f == NULL) { perror("fopen"); return 1; }

    size_t written = fwrite(numbers, sizeof numbers[0], n, f);
    printf("wrote %zu of %zu items\n", written, n);
    fclose(f);

    int loaded[5] = {0};
    f = fopen("numbers.bin", "rb");
    if (f == NULL) { perror("fopen"); return 1; }

    size_t got = fread(loaded, sizeof loaded[0], 5, f);
    printf("read %zu items:", got);
    for (size_t i = 0; i < got; i++) printf(" %d", loaded[i]);
    printf("\n");
    fclose(f);
    return 0;
}
// Output:
// wrote 5 of 5 items
// read 5 items: 10 20 30 40 50
```

The resulting `numbers.bin` is exactly 20 bytes. The same data as text
(`"10 20 30 40 50\n"`) would be 15 bytes here, but binary wins decisively as
values get larger, and it never needs parsing.

## Writing and reading structs

This is where binary I/O earns its keep — a whole record in one call:

```c
#include <stdio.h>
#include <string.h>

typedef struct {
    int    id;
    char   name[32];
    double price;
} Product;

int main(void) {
    Product items[3] = {
        {1, "keyboard", 49.95},
        {2, "monitor",  219.00},
        {3, "cable",     7.25}
    };

    FILE *f = fopen("products.dat", "wb");
    if (!f) { perror("open for write"); return 1; }
    if (fwrite(items, sizeof(Product), 3, f) != 3) {
        fprintf(stderr, "short write\n");
        fclose(f);
        return 1;
    }
    fclose(f);

    // Read them back one at a time
    f = fopen("products.dat", "rb");
    if (!f) { perror("open for read"); return 1; }

    Product p;
    while (fread(&p, sizeof p, 1, f) == 1) {
        printf("#%d %-10s $%.2f\n", p.id, p.name, p.price);
    }
    fclose(f);
    return 0;
}
// Output:
// #1 keyboard   $49.95
// #2 monitor    $219.00
// #3 cable      $7.25
```

`while (fread(&p, sizeof p, 1, f) == 1)` is the idiomatic record-reading loop:
it stops cleanly at EOF, and a partial record (a truncated file) also fails the
test instead of silently producing half-garbage.

Note **why this works only for this struct**: `Product` contains a fixed-size
`char name[32]`, not a `char *`. If it held a pointer, `fwrite` would faithfully
save the *address* — meaningless in any other process. Structs containing
pointers cannot be dumped and reloaded directly; you have to serialize what they
point to.

## Random access: `fseek`, `ftell`, `rewind`

Fixed-size records give you O(1) access to any record:

```c
#include <stdio.h>

typedef struct { int id; char name[32]; double price; } Product;

// Read record number `index` (0-based) without reading the ones before it.
int read_record(const char *path, long index, Product *out) {
    FILE *f = fopen(path, "rb");
    if (!f) return 0;

    if (fseek(f, index * (long)sizeof(Product), SEEK_SET) != 0) {
        fclose(f);
        return 0;
    }
    int ok = (fread(out, sizeof *out, 1, f) == 1);
    fclose(f);
    return ok;
}

// Overwrite record `index` in place -- note the "rb+" mode.
int update_price(const char *path, long index, double new_price) {
    FILE *f = fopen(path, "rb+");
    if (!f) return 0;

    Product p;
    if (fseek(f, index * (long)sizeof p, SEEK_SET) != 0 ||
        fread(&p, sizeof p, 1, f) != 1) {
        fclose(f);
        return 0;
    }

    p.price = new_price;
    fseek(f, index * (long)sizeof p, SEEK_SET);   // rewind to the record start
    int ok = (fwrite(&p, sizeof p, 1, f) == 1);
    fclose(f);
    return ok;
}

int main(void) {
    Product p;
    if (read_record("products.dat", 1, &p))
        printf("record 1: %s $%.2f\n", p.name, p.price);

    update_price("products.dat", 1, 199.99);

    if (read_record("products.dat", 1, &p))
        printf("record 1: %s $%.2f\n", p.name, p.price);
    return 0;
}
// Output:
// record 1: monitor $219.00
// record 1: monitor $199.99
```

`fseek(f, offset, whence)` takes one of three origins:

| `whence` | Offset measured from |
|----------|----------------------|
| `SEEK_SET` | the beginning of the file |
| `SEEK_CUR` | the current position |
| `SEEK_END` | the end of the file (use a negative offset to go back) |

`ftell(f)` reports the current byte offset, and `rewind(f)` is shorthand for
seeking to 0 and clearing the error flags.

### Counting records with `fseek` + `ftell`

```c
#include <stdio.h>

long record_count(const char *path, size_t record_size) {
    FILE *f = fopen(path, "rb");
    if (!f) return -1;

    fseek(f, 0, SEEK_END);      // jump to the end
    long bytes = ftell(f);      // ...and ask where that is
    fclose(f);

    if (bytes < 0) return -1;
    return bytes / (long)record_size;
}
```

One subtlety with `"rb+"`: the C standard requires a `fseek`, `fflush`, or
`rewind` **between** a read and a write on the same stream (and vice versa).
Skipping it is undefined behavior, even though it often appears to work.

## Portability: endianness and struct layout

The bytes you write are your machine's bytes. Two things can differ elsewhere:

- **Endianness.** `int 1` is `01 00 00 00` on x86/ARM (little-endian) and
  `00 00 00 01` on a big-endian machine. Same file, different number.
- **Padding.** As shown in [Module 4](04-structs-unions-advanced.md), a struct's
  internal padding is compiler- and platform-dependent, so `sizeof(Product)`
  may not match across builds.

```c
#include <stdio.h>

int main(void) {
    unsigned int x = 1;
    unsigned char *b = (unsigned char *)&x;
    printf("%s-endian (first byte = %u)\n",
           b[0] == 1 ? "little" : "big", b[0]);
    return 0;
}
// Output on x86-64 / Apple Silicon:
// little-endian (first byte = 1)
```

For files that only your program on your machine reads — caches, save files,
scratch data — raw struct dumps are perfectly fine and delightfully simple. For
anything that crosses machines, write **field by field in a defined byte order**
(or use a format like JSON, CBOR, or Protocol Buffers). A common middle ground
is to start every binary file with a small header containing a magic number and
a version:

```c
typedef struct {
    char     magic[4];      // "INVT"
    uint32_t version;       // bump when the record layout changes
    uint32_t record_count;
} FileHeader;
```

Reading a file whose magic doesn't match then fails loudly instead of
misinterpreting the bytes — and version numbers let old files stay readable.

## Exercise

Extend the `Product` example into a small binary database. Write
`int append_product(const char *path, const Product *p)` using `"ab"`,
`long count_products(const char *path)` using `fseek`/`ftell`, and
`int delete_product(const char *path, long index)` that copies every record
except `index` into a temporary file and then replaces the original. Add a
`FileHeader` with magic `"PROD"` and version `1` at the start of the file, and
make every read verify it before trusting the rest. Confirm your file size
matches `sizeof(FileHeader) + count * sizeof(Product)` with `ls -l`.
