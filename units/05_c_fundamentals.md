# Unit 05: C Language Fundamentals

## Overview

C is the native language of Linux. The kernel is written in C. Most system libraries are written in C. Understanding C means understanding how the machine actually works — there is almost nothing hidden between your source code and the CPU instructions it produces. This unit covers the core language: types, variables, operators, control flow, functions, arrays, and strings. By the end you will be writing real programs.

## Prerequisites

- Unit 02 (The Linux Environment) — you can navigate the terminal, use man pages, and run commands

## Learning Objectives

- Declare and use variables of the basic C types
- Write functions with parameters and return values
- Use loops and conditionals to control program flow
- Understand C strings as null-terminated character arrays
- Read and write from stdin/stdout using `printf` and `scanf`
- Compile and run a multi-function C program

## Reading / Resources

- `man 3 printf` — format string reference
- `man 3 scanf`
- The C Programming Language by Kernighan & Ritchie (K&R) — Chapters 1–4 (the definitive C reference)
- C Programming: A Modern Approach by K.N. King — more beginner-friendly than K&R

## Concepts

### Your First Program

```c
#include <stdio.h>

int main(void) {
    printf("Hello, Linux!\n");
    return 0;
}
```

Save as `hello.c`, compile, and run:

```bash
gcc hello.c -o hello
./hello
```

`#include <stdio.h>` pulls in the standard I/O library declarations. `main` is the entry point — every C program starts here. `return 0` signals success to the shell.

### Basic Types

| Type | Size | Range |
|------|------|-------|
| `char` | 1 byte | -128 to 127 (or 0–255 unsigned) |
| `int` | 4 bytes | -2,147,483,648 to 2,147,483,647 |
| `long` | 4 or 8 bytes | platform-dependent |
| `float` | 4 bytes | ~7 decimal digits |
| `double` | 8 bytes | ~15 decimal digits |

For systems programming, prefer explicit-width types from `<stdint.h>`:
```c
#include <stdint.h>

uint8_t  byte_value;   // always 8 bits, unsigned
int32_t  counter;      // always 32 bits, signed
uint64_t address;      // always 64 bits, unsigned
```

### Variables and Assignment

```c
int x = 10;
int y;       // uninitialized — value is garbage, don't read it
y = 20;
x = x + y;  // x is now 30
```

Constants:
```c
const int MAX = 100;   // cannot be changed
#define BUFSIZE 4096   // preprocessor constant — no type, no scope
```

### Operators

```c
// Arithmetic
x + y    x - y    x * y    x / y    x % y   // modulo

// Comparison (result is 0 or 1)
x == y   x != y   x < y   x > y   x <= y   x >= y

// Logical
x && y   // AND
x || y   // OR
!x       // NOT

// Bitwise (essential in systems programming)
x & y    // AND
x | y    // OR
x ^ y    // XOR
~x       // NOT (complement)
x << n   // left shift  (multiply by 2^n)
x >> n   // right shift (divide by 2^n)
```

### Control Flow

**if/else:**
```c
if (x > 0) {
    printf("positive\n");
} else if (x < 0) {
    printf("negative\n");
} else {
    printf("zero\n");
}
```

**while:**
```c
int i = 0;
while (i < 10) {
    printf("%d\n", i);
    i++;
}
```

**for:**
```c
for (int i = 0; i < 10; i++) {
    printf("%d\n", i);
}
```

**switch:**
```c
switch (c) {
case 'a':
    printf("alpha\n");
    break;
case 'b':
    printf("beta\n");
    break;
default:
    printf("other\n");
}
```

### Functions

```c
// Declaration (prototype) — tells the compiler the function exists
int add(int a, int b);

// Definition
int add(int a, int b) {
    return a + b;
}

// Call
int result = add(3, 4);  // result = 7
```

C passes arguments **by value** — the function gets a copy. Modifying a parameter does not affect the caller's variable.

```c
void increment(int x) {
    x++;  // modifies the local copy only
}

int n = 5;
increment(n);
// n is still 5
```

To modify the caller's variable, you need a pointer (Unit 07).

### Arrays

```c
int numbers[5] = {10, 20, 30, 40, 50};

numbers[0]  // 10 (zero-indexed)
numbers[4]  // 50

// Iterate
for (int i = 0; i < 5; i++) {
    printf("%d\n", numbers[i]);
}
```

Arrays in C do not know their own length. You must track the length yourself. Accessing out-of-bounds is **undefined behavior** — it will not crash reliably, which makes it dangerous.

### Strings

C strings are arrays of `char` terminated by a null byte (`'\0'`):

```c
char name[6] = {'h', 'e', 'l', 'l', 'o', '\0'};
char name[]  = "hello";  // same thing, compiler adds '\0'
```

String functions from `<string.h>`:

```c
#include <string.h>

strlen(s)           // length (not counting '\0')
strcpy(dst, src)    // copy string — DANGEROUS if dst is too small
strncpy(dst, src, n)// safer copy with length limit
strcat(dst, src)    // concatenate — also dangerous
strcmp(a, b)        // compare: 0 if equal, <0 or >0 otherwise
strstr(haystack, needle)  // find substring
```

**Warning:** `strcpy` and `strcat` are common sources of buffer overflow bugs. Always use the `n`-variants or snprintf.

### printf and scanf

```c
printf("int: %d, float: %.2f, string: %s, char: %c\n", 42, 3.14, "hi", 'x');

int age;
scanf("%d", &age);  // & gives scanf the address to write to (pointer — see Unit 07)
```

Common format specifiers:

| Specifier | Type |
|-----------|------|
| `%d` | `int` |
| `%u` | `unsigned int` |
| `%ld` | `long` |
| `%lu` | `unsigned long` |
| `%f` | `float`/`double` |
| `%s` | `char *` (string) |
| `%c` | `char` |
| `%p` | pointer (address) |
| `%x` | hex integer |
| `%zu` | `size_t` |

## Exercises

### Exercise 1: Hello World, Extended

Modify hello.c to ask the user for their name and print a personalized greeting.

```c
// Expected output:
// Enter your name: Alice
// Hello, Alice! Welcome to Linux native development.
```

### Exercise 2: FizzBuzz

Print numbers 1 through 100. For multiples of 3, print "Fizz" instead. For multiples of 5, print "Buzz". For multiples of both, print "FizzBuzz".

### Exercise 3: String Functions Without stdlib

Implement these functions yourself (do not use `<string.h>`):

```c
int my_strlen(const char *s);
void my_strcpy(char *dst, const char *src);
int my_strcmp(const char *a, const char *b);
```

Write a `main` that tests each one.

### Exercise 4: Number to Words

Write a function `void print_words(int n)` that prints the English word for numbers 0–19. For all others, print the digits. Call it for all numbers 0–25.

### Exercise 5: Histogram

Read up to 20 integers from the user (stop on EOF or when 20 are entered). Print a histogram with asterisks representing each value, scaled so the largest value = 20 asterisks.

```
Value 1 (42): ******************** 
Value 2 (21): **********
Value 3 (10): *****
```

## What Comes Next

Unit 06 covers compilation in depth — how gcc turns your `.c` file into a runnable binary, what Makefiles are, and how to manage multi-file projects. Unit 07 introduces pointers, which unlock the rest of systems programming.
