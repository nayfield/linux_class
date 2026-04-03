# Unit 17: Testing in C

## Overview

C has a reputation for being hard to test, but the difficulty is mostly a design problem: code that is tightly coupled to global state, file I/O, or specific platforms resists testing. Code written with testability in mind — small functions, clear interfaces, explicit dependencies — tests cleanly and catches bugs before they reach production. This unit covers how to structure C code for testing, how to use a test framework (Unity), how to measure coverage with gcov, and how to integrate testing into your build system. By the end, every project you ship will have a test suite.

## Prerequisites

- Unit 06 (Compilation & Build Tools) — Makefiles, object files, multiple compilation units
- Unit 07 (Pointers & Memory Management) — valgrind, ASan (tests should run clean under both)
- Unit 08 (Data Structures in C) — you have non-trivial code worth testing

## Learning Objectives

- Write unit tests in C using the Unity test framework
- Structure C code to be testable (separation of concerns, seams for injection)
- Use test doubles (stubs, fakes) to isolate units under test
- Measure code coverage with gcov and lcov
- Integrate tests into a Makefile and CMake build
- Understand the difference between unit, integration, and system tests
- Run tests automatically with sanitizers enabled

## Reading / Resources

- Unity test framework: https://github.com/ThrowTheSwitch/Unity
- `man 1 gcov`
- Test Driven Development for Embedded C by James Grenning (best C TDD book)
- CMocka: https://cmocka.org/ (alternative framework, supports mock objects)

## Concepts

### Why Testing C Is Different

In Python, you import a module and call functions. In C, the challenges are:
- **No exceptions** — errors come back as return values or errno; tests must check both
- **No runtime type info** — test frameworks use macros, not reflection
- **Global state** — functions that read globals or call `time()` are hard to test
- **Side effects** — functions that open files, send packets, or fork processes need seams
- **Memory ownership** — tests must not leak

The solution is not a better test framework. It is better code design.

### Writing Testable C

**The rule**: a function is testable if it takes all its inputs as parameters and returns its output (or writes to an output parameter). Side effects should be at the edges of the system.

```c
// HARD to test: reads global state, calls time()
int is_session_expired(void) {
    return time(NULL) > g_session.expires_at;
}

// EASY to test: all inputs as parameters
int is_session_expired(time_t now, time_t expires_at) {
    return now > expires_at;
}
```

```c
// HARD to test: opens a real file
int count_lines(const char *filename) {
    FILE *f = fopen(filename, "r");
    // ...
}

// EASY to test: accepts a FILE* (can pass any stream, including test streams)
int count_lines(FILE *f) {
    // ...
}
```

**Seams**: places where you can substitute a test implementation for a real one without changing the code under test. In C, the main seam mechanisms are:
- **Function pointer injection**: pass the dependency as a parameter
- **Compile-time substitution**: link a test double `.o` instead of the real one
- **Preprocessor**: `#define real_func test_fake` (last resort)

### Unity Test Framework

Unity is a minimal, single-file C test framework widely used in embedded and systems code.

Install (just copy the source files):
```bash
# Option 1: vendored
curl -o unity.c https://raw.githubusercontent.com/ThrowTheSwitch/Unity/master/src/unity.c
curl -o unity.h https://raw.githubusercontent.com/ThrowTheSwitch/Unity/master/src/unity.h
curl -o unity_internals.h https://raw.githubusercontent.com/ThrowTheSwitch/Unity/master/src/unity_internals.h

# Option 2: package manager
apt install libunity-dev   # Ubuntu/Debian (may vary)
```

A test file:
```c
#include "unity.h"
#include "math_utils.h"

void setUp(void) {}     // called before each test
void tearDown(void) {}  // called after each test

void test_gcd_basic(void) {
    TEST_ASSERT_EQUAL_INT(4, gcd(8, 12));
    TEST_ASSERT_EQUAL_INT(1, gcd(7, 13));
    TEST_ASSERT_EQUAL_INT(6, gcd(6, 6));
}

void test_gcd_zero(void) {
    TEST_ASSERT_EQUAL_INT(5, gcd(5, 0));
    TEST_ASSERT_EQUAL_INT(5, gcd(0, 5));
}

void test_lcm(void) {
    TEST_ASSERT_EQUAL_INT(12, lcm(4, 6));
    TEST_ASSERT_EQUAL_INT(0,  lcm(0, 5));
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_gcd_basic);
    RUN_TEST(test_gcd_zero);
    RUN_TEST(test_lcm);
    return UNITY_END();
}
```

**Key Unity assertions:**

```c
TEST_ASSERT_TRUE(condition)
TEST_ASSERT_FALSE(condition)
TEST_ASSERT_NULL(pointer)
TEST_ASSERT_NOT_NULL(pointer)

TEST_ASSERT_EQUAL_INT(expected, actual)
TEST_ASSERT_EQUAL_UINT(expected, actual)
TEST_ASSERT_EQUAL_FLOAT(expected, actual, delta)
TEST_ASSERT_EQUAL_STRING(expected, actual)

TEST_ASSERT_EQUAL_INT_ARRAY(expected, actual, num_elements)
TEST_ASSERT_EQUAL_MEMORY(expected, actual, len)

TEST_FAIL_MESSAGE("custom message")
TEST_IGNORE_MESSAGE("not implemented yet")
```

Exit code 0 = all tests pass; non-zero = failures (important for CI).

### Test Doubles in C

When a function depends on something hard to test (a real clock, a real network), substitute a **test double**:

**Stub**: returns hardcoded values
```c
// In production: real_time.c
time_t get_current_time(void) { return time(NULL); }

// In test: fake_time.c (linked instead)
static time_t g_fake_time = 1000000;
time_t get_current_time(void) { return g_fake_time; }
void   set_fake_time(time_t t) { g_fake_time = t; }
```

**Function pointer injection**: more flexible, can switch at runtime
```c
typedef int (*send_fn)(int fd, const void *buf, size_t len, int flags);

typedef struct {
    send_fn send;
    // ... other dependencies
} NetworkCtx;

int http_send_response(NetworkCtx *ctx, int fd, int status) {
    const char *msg = "HTTP/1.0 200 OK\r\n\r\n";
    return ctx->send(fd, msg, strlen(msg), 0);
}

// In test:
static int fake_send_called = 0;
static int fake_send(int fd, const void *buf, size_t len, int flags) {
    fake_send_called++;
    return len;  // pretend everything was sent
}

void test_http_send_response(void) {
    NetworkCtx ctx = { .send = fake_send };
    fake_send_called = 0;
    int result = http_send_response(&ctx, 42, 200);
    TEST_ASSERT_EQUAL_INT(1, fake_send_called);
    TEST_ASSERT_TRUE(result > 0);
}
```

### Project Structure for Testing

```
project/
├── src/
│   ├── math_utils.c
│   ├── math_utils.h
│   ├── hash_table.c
│   └── hash_table.h
├── tests/
│   ├── unity.c           (vendored)
│   ├── unity.h
│   ├── unity_internals.h
│   ├── test_math_utils.c
│   └── test_hash_table.c
├── Makefile
└── CMakeLists.txt
```

### Makefile for Tests

```makefile
CC      = gcc
CFLAGS  = -Wall -Wextra -g -std=c11 -fsanitize=address
LDFLAGS = -fsanitize=address

SRC     = $(wildcard src/*.c)
TESTS   = $(wildcard tests/test_*.c)

# Build and run all tests
.PHONY: test
test: $(TESTS:.c=)
	@for t in $^; do echo "--- $$t ---"; ./$$t; done

tests/test_%: tests/test_%.c src/%.c tests/unity.c
	$(CC) $(CFLAGS) $^ -o $@ $(LDFLAGS)

clean:
	rm -f tests/test_*[^.c]
```

### Code Coverage with gcov

`gcov` instruments your binary to track which lines are executed:

```bash
# Compile with coverage flags
gcc -Wall -g --coverage src/math_utils.c tests/test_math_utils.c tests/unity.c -o test_math

# Run the tests
./test_math

# Generate coverage report
gcov src/math_utils.c
# outputs: math_utils.c.gcov

# HTML report with lcov
lcov --capture --directory . --output-file coverage.info
genhtml coverage.info --output-directory coverage_html
# Open coverage_html/index.html in a browser
```

Coverage is a tool for finding **untested code paths**, not a goal in itself. 100% line coverage does not mean your tests are good — it means every line ran at least once.

### CMake Integration

```cmake
cmake_minimum_required(VERSION 3.15)
project(myproject C)
set(CMAKE_C_STANDARD 11)

enable_testing()

# Library under test
add_library(math_utils src/math_utils.c)
target_include_directories(math_utils PUBLIC src)

# Test executable
add_executable(test_math_utils
    tests/test_math_utils.c
    tests/unity.c)
target_link_libraries(test_math_utils math_utils)
target_compile_options(test_math_utils PRIVATE -fsanitize=address)
target_link_options(test_math_utils PRIVATE -fsanitize=address)

# Register with CTest
add_test(NAME math_utils COMMAND test_math_utils)
```

Run tests:
```bash
cmake -B build && cmake --build build
cd build && ctest --output-on-failure
```

### Integration vs Unit Tests

| Type | Tests | Speed | Dependencies |
|------|-------|-------|-------------|
| Unit | One function in isolation | Fast (ms) | Test doubles |
| Integration | Multiple modules together | Medium | Some real deps |
| System | Full program end-to-end | Slow | Everything real |

Write mostly unit tests (fast, isolated, precise). Write integration tests for the boundaries between modules. Write system tests for critical user-facing behaviors.

For C systems code, a practical system test is often a shell script:
```bash
#!/bin/bash
set -euo pipefail
./server &
SERVER_PID=$!
sleep 0.1
result=$(echo "ping" | nc -q1 localhost 9000)
kill $SERVER_PID
[ "$result" = "ping" ] || { echo "FAIL: expected ping, got $result"; exit 1; }
echo "PASS"
```

### Test-Driven Development in C

TDD in C is practical and effective:

1. Write a failing test for the next small piece of behavior
2. Write the minimum code to make it pass
3. Refactor if needed, keeping tests green

```c
// Step 1: write the test first
void test_stack_push_pop(void) {
    Stack *s = stack_new();
    stack_push(s, 10);
    stack_push(s, 20);
    TEST_ASSERT_EQUAL_INT(20, stack_pop(s));
    TEST_ASSERT_EQUAL_INT(10, stack_pop(s));
    stack_free(s);
}
// Compile fails: no stack_new, stack_push, stack_pop

// Step 2: write the header
// stack.h: Stack *stack_new(void); void stack_push(...); int stack_pop(...);

// Step 3: write the minimal implementation
// Step 4: test passes — now refactor if needed
```

## Exercises

### Exercise 1: Test Your Data Structures

Go back to Unit 08 and write a comprehensive Unity test suite for your hash table implementation:
1. Test insert and lookup for 0, 1, and many items
2. Test collision handling (force collisions by using a tiny table)
3. Test deletion — verify deleted keys are not found
4. Test resize — verify all items survive
5. Test iteration — verify all items are visited exactly once
6. Run with `--coverage` and achieve >90% line coverage
7. Run under AddressSanitizer — zero errors

### Exercise 2: Test-Drive a Stack

Implement a stack data structure using strict TDD:
1. Write `test_stack_empty` first — `stack_is_empty` returns 1 for new stack
2. Write `test_stack_push_pop` — basic push/pop works
3. Write `test_stack_underflow` — pop on empty returns an error indicator, not a crash
4. Write `test_stack_many` — push 1000 items, pop all, verify order
5. Write the implementation only after the tests are written

### Exercise 3: Testing I/O Code

Take the `count_lines` function and make it testable by accepting a `FILE*`:

```c
// Implement and test:
size_t count_lines(FILE *f);
size_t count_words(FILE *f);
size_t count_bytes(FILE *f);
```

Use `fmemopen` to create a `FILE*` backed by a string literal (no temp file needed):
```c
const char *input = "line one\nline two\nline three\n";
FILE *f = fmemopen((void*)input, strlen(input), "r");
TEST_ASSERT_EQUAL_INT(3, count_lines(f));
fclose(f);
```

### Exercise 4: Coverage Analysis

Pick the most complex module you have written so far (hash table or BST from Unit 08). Run gcov/lcov and look at the HTML report:
1. Identify any lines with 0 coverage
2. Write tests specifically to cover those branches
3. Achieve >95% line coverage
4. Take a screenshot or save the report

Reflect: were the uncovered paths error-handling paths? Edge cases? Dead code?

### Exercise 5: End-to-End Test Script

Write a shell script that tests your TCP echo server from Unit 15 end-to-end:
1. Start the server in the background
2. Connect with `nc` and send 5 messages, verify each is echoed
3. Verify the server handles 10 concurrent connections
4. Kill the server gracefully with `SIGTERM`
5. Verify the server process exited cleanly (exit code 0)
6. Report PASS or FAIL with a clear message

Integrate this script into your Makefile as `make test-integration`.

## What Comes Next

Unit 18 (Packaging & Deployment) is the final unit. With a test suite in place, you can be confident that your packaged software works correctly, your systemd service restarts cleanly, and your Docker container behaves as expected.
