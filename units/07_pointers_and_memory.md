# Unit 07: Pointers & Memory Management

## Overview

Pointers are the feature that distinguishes C from higher-level languages. They give you direct access to memory addresses, enable dynamic data structures, and are the mechanism by which functions can modify the caller's data. Mastering pointers is the single biggest step in becoming a Linux native developer. This unit also covers dynamic memory allocation (`malloc`/`free`) and the common memory bugs that crash production systems. After this unit, revisit Unit 01 — the Drepper paper will make much more sense.

## Prerequisites

- Unit 05 (C Language Fundamentals)
- Unit 06 (Compilation & Build Tools) — you need `gcc -g` and valgrind for the exercises

## Learning Objectives

- Declare, dereference, and perform arithmetic on pointers
- Pass pointers to functions to modify the caller's data
- Understand the relationship between arrays and pointers
- Allocate and free heap memory correctly with `malloc`, `calloc`, `realloc`, and `free`
- Draw the memory layout of a running process (stack, heap, code, data segments)
- Identify and avoid use-after-free, double-free, memory leaks, and buffer overflows
- Use valgrind to detect memory errors

## Reading / Resources

- K&R — Chapter 5 (Pointers and Arrays)
- `man 3 malloc`
- `man 3 free`
- `man 1 valgrind`

## Concepts

### What Is a Pointer?

A pointer is a variable that holds a **memory address**. Every variable lives at some address in memory. A pointer stores that address.

```c
int x = 42;
int *p = &x;    // p holds the address of x

printf("value of x: %d\n", x);     // 42
printf("address of x: %p\n", &x);  // e.g. 0x7ffe1234
printf("p holds: %p\n", p);        // same address
printf("*p (dereference): %d\n", *p); // 42

*p = 100;     // write through the pointer
printf("x is now: %d\n", x);  // 100
```

The `&` operator gives you the address of a variable. The `*` operator **dereferences** a pointer — it gives you the value at the address the pointer holds.

### Pointers and Functions

To modify a variable from inside a function, pass a pointer to it:

```c
void increment(int *p) {
    (*p)++;  // dereference and increment
}

int n = 5;
increment(&n);
printf("%d\n", n);  // 6
```

### Pointer Arithmetic

You can add integers to pointers. The arithmetic is in units of the pointed-to type's size:

```c
int arr[5] = {10, 20, 30, 40, 50};
int *p = arr;    // points to arr[0]

printf("%d\n", *p);      // 10
printf("%d\n", *(p+1));  // 20 — same as arr[1]
printf("%d\n", *(p+4));  // 50 — same as arr[4]

p++;             // advance to next int (4 bytes forward)
printf("%d\n", *p);  // 20
```

`arr[i]` is exactly equivalent to `*(arr + i)`. Arrays and pointers are deeply related in C.

### Arrays and Pointers

When you use an array name in most expressions, it **decays** to a pointer to its first element:

```c
int arr[5] = {1, 2, 3, 4, 5};
int *p = arr;    // no & needed — arr decays to &arr[0]

// These are all equivalent:
arr[2]
*(arr + 2)
p[2]
*(p + 2)
```

One critical difference: `sizeof(arr)` gives the total size of the array (20 bytes for 5 ints), but `sizeof(p)` gives the size of the pointer (8 bytes on 64-bit). When you pass an array to a function, it becomes a pointer and you lose the size information — always pass the length separately.

### Memory Layout of a Process

A running process has several memory regions:

```
High addresses
┌─────────────────┐
│   Stack         │  Local variables, function call frames
│   (grows down)  │  Automatic allocation/deallocation
├─────────────────┤
│   ...           │  (gap — unmapped)
├─────────────────┤
│   Heap          │  Dynamic allocation (malloc/free)
│   (grows up)    │
├─────────────────┤
│   BSS segment   │  Uninitialized global variables
├─────────────────┤
│   Data segment  │  Initialized global variables
├─────────────────┤
│   Text segment  │  Program code (read-only)
└─────────────────┘
Low addresses
```

```c
int global = 5;         // data segment
int uninit;             // BSS segment

int main(void) {
    int local = 10;     // stack
    int *heap = malloc(sizeof(int) * 100);  // heap
    free(heap);
    return 0;
}
```

Inspect your own process layout:
```bash
cat /proc/self/maps
```

### Dynamic Memory Allocation

The **stack** is automatic — locals are allocated when a function is entered and freed when it returns. The **heap** is manual — you allocate and free it explicitly.

```c
#include <stdlib.h>

// malloc: allocate n bytes, uninitialized
int *p = malloc(10 * sizeof(int));
if (p == NULL) {
    // allocation failed — always check
    perror("malloc");
    exit(1);
}

// calloc: allocate and zero-initialize
int *q = calloc(10, sizeof(int));

// realloc: resize an existing allocation
p = realloc(p, 20 * sizeof(int));  // grow to 20 ints
if (p == NULL) { /* handle error */ }

// free: release heap memory
free(p);
free(q);
p = NULL;  // good practice — prevents accidental use-after-free
q = NULL;
```

### Common Memory Bugs

These bugs corrupt memory silently and cause crashes or security vulnerabilities far from the source of the bug.

**Memory leak:** allocate but never free
```c
void leaky(void) {
    int *p = malloc(100);
    // forgot free(p) — memory is never returned
}
```

**Use-after-free:** access memory after freeing it
```c
int *p = malloc(sizeof(int));
*p = 42;
free(p);
printf("%d\n", *p);  // undefined behavior — p is a dangling pointer
```

**Double free:** free the same memory twice
```c
free(p);
free(p);  // undefined behavior — corrupts the heap allocator
```

**Buffer overflow:** write past the end of an allocation
```c
int *arr = malloc(5 * sizeof(int));
arr[10] = 99;  // write to memory you don't own
```

**Null pointer dereference:** dereference without checking
```c
int *p = malloc(sizeof(int));
// if malloc fails, p is NULL
*p = 42;  // crash if p is NULL
```

### Using Valgrind

Valgrind's `memcheck` tool detects all of the above at runtime:

```bash
gcc -Wall -g program.c -o program
valgrind --leak-check=full ./program
```

Valgrind will report:
- Memory leaks (bytes still allocated at exit)
- Invalid reads/writes (out of bounds, use-after-free)
- Use of uninitialized values
- Double frees

A clean program produces: `ERROR SUMMARY: 0 errors from 0 contexts`

### Strings Revisited

Now that you understand pointers, string handling makes more sense:

```c
char *s = "hello";        // pointer to string literal (read-only!)
char buf[6] = "hello";    // array copy (modifiable)

// Never do this:
char *s = "hello";
s[0] = 'H';  // segfault — string literals are in read-only memory
```

```c
// strlen returns the length, not counting '\0'
// The actual allocation must be length + 1
size_t len = strlen("hello");  // 5
char *copy = malloc(len + 1);
strcpy(copy, "hello");
free(copy);
```

## Exercises

### Exercise 1: Pointer Basics

Write a `swap` function that exchanges two integers using pointers:

```c
void swap(int *a, int *b);
```

Test it with several pairs. Verify by printing the values before and after.

### Exercise 2: Dynamic Array

Implement a growable integer array:

```c
typedef struct {
    int *data;
    size_t len;
    size_t capacity;
} IntArray;

void array_init(IntArray *arr);
void array_push(IntArray *arr, int value);  // doubles capacity when full
int  array_get(IntArray *arr, size_t i);
void array_free(IntArray *arr);
```

When `push` needs more space, use `realloc` to double the capacity. Run your program under valgrind and confirm zero errors.

### Exercise 3: Hunt the Bugs

This program has four memory bugs. Find and fix them all, then verify with valgrind:

```c
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

char *duplicate(const char *s) {
    char *copy = malloc(strlen(s));  // bug 1
    strcpy(copy, s);
    return copy;
}

int main(void) {
    char *a = duplicate("hello");
    char *b = duplicate("world");

    printf("%s %s\n", a, b);

    free(a);
    free(a);  // bug 2
    // bug 3: b is never freed

    int *arr = malloc(3 * sizeof(int));
    arr[3] = 99;  // bug 4
    free(arr);

    return 0;
}
```

### Exercise 4: Matrix Operations

Implement a heap-allocated 2D matrix:

```c
int **matrix_create(int rows, int cols);
void matrix_set(int **m, int row, int col, int value);
int  matrix_get(int **m, int row, int col);
void matrix_free(int **m, int rows);
void matrix_multiply(int **a, int **b, int **result, int n);
```

Revisit your answer from Unit 01 Exercise 3 — which traversal order is faster in `matrix_multiply`? Time it with large matrices (n=500).

### Exercise 5: Stack vs Heap Exploration

Run this program and observe `/proc/PID/maps` while it pauses:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int global = 42;

int main(void) {
    int stack_var = 10;
    int *heap_var = malloc(1024 * 1024);  // 1MB on heap

    printf("PID: %d\n", getpid());
    printf("global:    %p\n", (void*)&global);
    printf("stack_var: %p\n", (void*)&stack_var);
    printf("heap_var:  %p\n", (void*)heap_var);
    printf("main:      %p\n", (void*)main);

    printf("Press enter to continue...\n");
    getchar();

    free(heap_var);
    return 0;
}
```

In another terminal: `cat /proc/<PID>/maps`. Identify which memory region each address falls in.

## What Comes Next

Unit 08 builds on pointers directly — you will implement linked lists, hash tables, and trees from scratch. Unit 09 then uses pointers extensively for file I/O buffers, and Unit 14 (Debugging) teaches you to catch pointer bugs live with gdb.
