# 10 · Project — CLI Contact Book

A small end-to-end project combining everything from Level 1: structs, arrays,
strings, pointers, file I/O, and multi-file compilation.

## What you'll build

A command-line contact book that:

- Adds a contact (name + phone number)
- Lists all contacts
- Searches for a contact by name
- Saves everything to a text file so contacts persist between runs

## Project layout

```text
contacts/
    contacts.h
    contacts.c
    main.c
```

## contacts.h — shared declarations

```c
// contacts.h
#ifndef CONTACTS_H
#define CONTACTS_H

#define MAX_CONTACTS 100
#define NAME_LEN 50
#define PHONE_LEN 20

typedef struct {
    char name[NAME_LEN];
    char phone[PHONE_LEN];
} Contact;

int load_contacts(Contact contacts[], int max);
void save_contacts(const Contact contacts[], int count);
void add_contact(Contact contacts[], int *count, const char *name, const char *phone);
void list_contacts(const Contact contacts[], int count);
int find_contact(const Contact contacts[], int count, const char *name);

#endif
```

## contacts.c — implementation

```c
// contacts.c
#include <stdio.h>
#include <string.h>
#include "contacts.h"

int load_contacts(Contact contacts[], int max) {
    FILE *f = fopen("contacts.txt", "r");
    if (f == NULL) {
        return 0;   // no file yet -- start with zero contacts
    }

    int count = 0;
    while (count < max &&
           fscanf(f, "%49[^,],%19[^\n]\n", contacts[count].name, contacts[count].phone) == 2) {
        count++;
    }

    fclose(f);
    return count;
}

void save_contacts(const Contact contacts[], int count) {
    FILE *f = fopen("contacts.txt", "w");
    if (f == NULL) {
        printf("Error: could not save contacts.\n");
        return;
    }

    for (int i = 0; i < count; i++) {
        fprintf(f, "%s,%s\n", contacts[i].name, contacts[i].phone);
    }

    fclose(f);
}

void add_contact(Contact contacts[], int *count, const char *name, const char *phone) {
    if (*count >= MAX_CONTACTS) {
        printf("Contact book is full.\n");
        return;
    }

    strncpy(contacts[*count].name, name, NAME_LEN - 1);
    contacts[*count].name[NAME_LEN - 1] = '\0';

    strncpy(contacts[*count].phone, phone, PHONE_LEN - 1);
    contacts[*count].phone[PHONE_LEN - 1] = '\0';

    (*count)++;
}

void list_contacts(const Contact contacts[], int count) {
    if (count == 0) {
        printf("No contacts yet.\n");
        return;
    }

    for (int i = 0; i < count; i++) {
        printf("%d. %s - %s\n", i + 1, contacts[i].name, contacts[i].phone);
    }
}

int find_contact(const Contact contacts[], int count, const char *name) {
    for (int i = 0; i < count; i++) {
        if (strcmp(contacts[i].name, name) == 0) {
            return i;   // found -- return its index
        }
    }
    return -1;   // not found
}
```

## main.c — CLI logic

```c
// main.c
#include <stdio.h>
#include <string.h>
#include "contacts.h"

int main(void) {
    Contact contacts[MAX_CONTACTS];
    int count = load_contacts(contacts, MAX_CONTACTS);

    int choice;
    char name[NAME_LEN];
    char phone[PHONE_LEN];

    do {
        printf("\n1. Add  2. List  3. Search  4. Save & Quit\n> ");
        scanf("%d", &choice);
        getchar();   // consume the leftover newline from scanf

        switch (choice) {
            case 1:
                printf("Name: ");
                fgets(name, NAME_LEN, stdin);
                name[strcspn(name, "\n")] = '\0';   // strip trailing newline

                printf("Phone: ");
                fgets(phone, PHONE_LEN, stdin);
                phone[strcspn(phone, "\n")] = '\0';

                add_contact(contacts, &count, name, phone);
                printf("Added.\n");
                break;

            case 2:
                list_contacts(contacts, count);
                break;

            case 3:
                printf("Name to search: ");
                fgets(name, NAME_LEN, stdin);
                name[strcspn(name, "\n")] = '\0';

                int idx = find_contact(contacts, count, name);
                if (idx >= 0) {
                    printf("Found: %s - %s\n", contacts[idx].name, contacts[idx].phone);
                } else {
                    printf("Not found.\n");
                }
                break;

            case 4:
                save_contacts(contacts, count);
                printf("Saved. Goodbye!\n");
                break;

            default:
                printf("Unknown option.\n");
        }
    } while (choice != 4);

    return 0;
}
```

## Compiling and running

```bash
gcc -Wall -Wextra -o contacts main.c contacts.c
./contacts
```

```text
1. Add  2. List  3. Search  4. Save & Quit
> 1
Name: Ada Lovelace
Phone: 555-0100
Added.

1. Add  2. List  3. Search  4. Save & Quit
> 2
1. Ada Lovelace - 555-0100

1. Add  2. List  3. Search  4. Save & Quit
> 4
Saved. Goodbye!
```

## Stretch goals

- Add a `delete_contact` function and a matching menu option.
- Sort contacts alphabetically before listing (a simple bubble sort over the
  array works fine at this size).
- Add input validation so a phone number must be non-empty before `add_contact`
  is called.

Completing this project means you're ready for **Level 2 · Intermediate**.
