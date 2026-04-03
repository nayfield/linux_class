# Unit 06: Compilation & Build Tools

## Overview

Writing C code is only half the job. Understanding how that code becomes a running program — and how to manage that process reliably — is equally important. This unit covers the four stages of compilation, the essential gcc flags every Linux developer uses, and Makefiles as the foundation of build automation. By the end you will be able to manage a multi-file C project with a professional Makefile.

## Prerequisites

- Unit 05 (C Language Fundamentals) — can write and compile basic C programs

## Learning Objectives

- Explain the four stages of compilation (preprocess, compile, assemble, link)
- Use gcc flags for warnings, optimization, debugging, and output control
- Understand what object files and libraries are
- Write a Makefile with targets, rules, variables, and automatic dependency tracking
- Know when to reach for cmake vs make

## Reading / Resources

- `man 1 gcc` — full gcc option reference
- `man 1 make` — GNU make manual
- GNU Make Manual: https://www.gnu.org/software/make/manual/
- CMake documentation: https://cmake.org/cmake/help/latest/

## Concepts

### The Four Stages of Compilation

A C source file goes through four stages before becoming an executable:

```
source.c
    │
    ▼ (1) Preprocessing  — gcc -E
source.i        # macros expanded, #includes inlined
    │
    ▼ (2) Compilation    — gcc -S
source.s        # assembly language
    │
    ▼ (3) Assembly       — gcc -c
source.o        # machine code (object file)
    │
    ▼ (4) Linking        — gcc (ld)
a.out / program # executable
```

You can inspect each stage:

```bash
gcc -E hello.c -o hello.i    # stop after preprocessing
gcc -S hello.c -o hello.s    # stop after compiling to assembly
gcc -c hello.c -o hello.o    # stop after assembling to object file
gcc hello.o -o hello         # link object file into executable
```

Read the assembly output (`cat hello.s`) — even without knowing assembly, you will start to see how C maps to machine instructions.

### Essential gcc Flags

**Warnings — always use these:**
```bash
-Wall       # enable most important warnings
-Wextra     # enable additional warnings
-Werror     # treat warnings as errors (strict mode)
-pedantic   # enforce strict ISO C compliance
```

**Optimization:**
```bash
-O0    # no optimization (default) — best for debugging
-O1    # light optimization
-O2    # standard release optimization
-O3    # aggressive optimization (can increase binary size)
-Os    # optimize for size
```

**Debugging:**
```bash
-g     # include debug symbols (required for gdb)
-g3    # include even macro definitions in debug info
```

**Other important flags:**
```bash
-o name           # set output filename
-std=c11          # use C11 standard (or c99, c17, gnu11…)
-I/path/to/headers  # add header search directory
-L/path/to/libs   # add library search directory
-lname            # link against libname.so or libname.a
-D MACRO          # define a preprocessor macro
-DDEBUG=1         # define MACRO with a value
```

A typical development build:
```bash
gcc -Wall -Wextra -g -std=c11 -o myprogram main.c utils.c
```

A typical release build:
```bash
gcc -Wall -Wextra -O2 -std=c11 -o myprogram main.c utils.c
```

### Object Files and Linking

When you have multiple `.c` files, compile each to an object file, then link them together:

```bash
gcc -c -Wall -g main.c -o main.o
gcc -c -Wall -g utils.c -o utils.o
gcc main.o utils.o -o myprogram
```

This is more efficient: if only `utils.c` changes, you only recompile `utils.o`.

### Makefiles

A `Makefile` automates the build process. Make only rebuilds what has changed.

**Basic structure:**

```makefile
target: dependencies
	recipe command
```

Note: the recipe **must** be indented with a **tab**, not spaces.

**A minimal Makefile:**

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -g -std=c11

myprogram: main.o utils.o
	$(CC) main.o utils.o -o myprogram

main.o: main.c utils.h
	$(CC) $(CFLAGS) -c main.c -o main.o

utils.o: utils.c utils.h
	$(CC) $(CFLAGS) -c utils.c -o utils.o

clean:
	rm -f *.o myprogram

.PHONY: clean
```

Run with `make` (builds default target) or `make clean`.

**Automatic variables:**

| Variable | Meaning |
|----------|---------|
| `$@` | The target name |
| `$<` | The first dependency |
| `$^` | All dependencies |
| `$*` | The stem (matched by %) |

**Pattern rules — a professional Makefile:**

```makefile
CC      = gcc
CFLAGS  = -Wall -Wextra -Werror -g -std=c11
LDFLAGS =
TARGET  = myprogram
SRCS    = $(wildcard *.c)
OBJS    = $(SRCS:.c=.o)

$(TARGET): $(OBJS)
	$(CC) $(LDFLAGS) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(TARGET)

.PHONY: clean
```

### Automatic Dependency Generation

The Makefile above has a problem: it does not know which header files each `.c` file includes. If you change a header, make will not rebuild the affected `.o` files.

gcc can generate dependency information automatically:

```makefile
DEPFLAGS = -MMD -MP
DEPS     = $(OBJS:.o=.d)

%.o: %.c
	$(CC) $(CFLAGS) $(DEPFLAGS) -c $< -o $@

-include $(DEPS)
```

The `-MMD` flag makes gcc generate a `.d` file alongside each `.o`, containing the correct dependency rules.

### Introduction to CMake

For larger projects, CMake is the standard. CMake generates Makefiles (or Ninja files) for you.

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.10)
project(myprogram C)
set(CMAKE_C_STANDARD 11)

add_executable(myprogram main.c utils.c)
target_compile_options(myprogram PRIVATE -Wall -Wextra)
```

Build:
```bash
mkdir build && cd build
cmake ..
make
```

Use cmake for projects with: multiple subdirectories, third-party library dependencies (via `find_package`), or cross-platform builds. Stick with make for simple single-directory projects.

## Exercises

### Exercise 1: Inspect the Compilation Stages

Take any `.c` file from Unit 05 and run it through each compilation stage:

```bash
gcc -E source.c -o source.i
gcc -S source.c -o source.s
gcc -c source.c -o source.o
gcc source.o -o program
```

Read each output file. For `source.s`, find your `main` function in the assembly.

### Exercise 2: Warning Discipline

Write this deliberately buggy program and see how many warnings gcc catches:

```c
#include <stdio.h>

int foo(int x) {
    // missing return
}

int main() {
    int x;
    printf("%d\n", x);  // uninitialized variable
    int arr[3] = {1, 2, 3};
    printf("%d\n", arr[5]);  // out of bounds (may not warn)
    foo(42);
    return 0;
}
```

Compile with progressively stricter flags:
```bash
gcc buggy.c -o buggy                        # no warnings
gcc -Wall buggy.c -o buggy                  # with -Wall
gcc -Wall -Wextra buggy.c -o buggy          # with -Wextra
gcc -Wall -Wextra -Werror buggy.c -o buggy  # fail on warnings
```

### Exercise 3: Multi-File Project with Makefile

Create a project with this structure:

```
project/
├── Makefile
├── main.c
├── math_utils.c
└── math_utils.h
```

`math_utils.h` declares:
```c
int gcd(int a, int b);
int lcm(int a, int b);
```

`math_utils.c` implements them. `main.c` includes `math_utils.h` and calls them.

Write a Makefile using pattern rules that:
- Compiles each `.c` to a `.o`
- Links the final binary
- Has a `clean` target
- Uses `-Wall -Wextra -g`

Verify that changing only `math_utils.c` only recompiles `math_utils.o`, not `main.o`.

### Exercise 4: CMake Build

Convert the Exercise 3 project to use CMake. Create a `CMakeLists.txt` and build in a `build/` subdirectory.

## What Comes Next

Unit 07 introduces pointers — the most important and most misunderstood feature of C. Everything from here forward depends on understanding them.
