# Unit 08a: Pointer Arithmetic for Data Processing

## Overview

Unit 07 introduced pointer arithmetic basics. This unit goes deeper — focusing on using pointers to navigate sequences of data and reference neighboring values. This is a fundamental pattern in compression algorithms, signal processing, and any code that computes over streams. When you're looking at `data[i]`, you often need `data[i-1]` and `data[i+1]` — and doing this with raw pointers is both faster and more natural than indexing. By the end of this unit, you'll implement delta encoding, a real compression technique used in network protocols and time-series databases.

## Prerequisites

- Unit 07 (Pointers & Memory Management) — pointer basics, malloc/free, pointer arithmetic fundamentals
- Unit 08 (Data Structures in C) — structs, working with dynamically allocated buffers

## Learning Objectives

- Navigate forward and backward through memory using pointer arithmetic
- Access previous and next elements relative to a pointer's current position
- Understand pointer subtraction and the `ptrdiff_t` type
- Implement delta encoding and decoding using pointer-based traversal
- Recognize boundary conditions when accessing neighbors in a sequence

## Reading / Resources

- K&R — Chapter 5, Section 5.4 (Address Arithmetic)
- RFC 3284 (VCDIFF) — a real delta-encoding compression format
- `man 3 memcpy`, `man 3 memmove`

## Concepts

### Pointers as Cursors

Think of a pointer as a cursor into a block of data. You can move it forward, move it backward, and peek at neighbors — all without computing an index.

```c
int data[] = {100, 104, 108, 102, 110, 115};
int n = sizeof(data) / sizeof(data[0]);

int *cursor = &data[2];  // cursor points at 108

printf("current:  %d\n", *cursor);       // 108
printf("previous: %d\n", *(cursor - 1)); // 104
printf("next:     %d\n", *(cursor + 1)); // 102
```

`*(cursor - 1)` reads the element before the cursor. `*(cursor + 1)` reads the element after. The compiler handles the actual byte offsets — for `int *`, each `+1` moves 4 bytes forward.

### Walking a Sequence with Neighbor Access

When processing data, you frequently need the previous value. Here's the pattern:

```c
void print_differences(int *data, int n) {
    int *prev = data;
    int *curr = data + 1;

    while (curr < data + n) {
        printf("value: %3d  diff from previous: %+d\n",
               *curr, *curr - *prev);
        prev = curr;
        curr++;
    }
}
```

This two-pointer approach is clean and explicit. `prev` trails `curr` by one position as they walk through the array together.

An equivalent approach using a single pointer:

```c
void print_differences_v2(int *data, int n) {
    for (int *p = data + 1; p < data + n; p++) {
        printf("value: %3d  diff from previous: %+d\n",
               *p, *p - *(p - 1));
    }
}
```

Both are valid. The two-pointer version is easier to read when the relationship between neighbors gets complex.

### Boundary Safety

When accessing `*(p - 1)` or `*(p + 1)`, you must ensure you're within bounds. Reading before the start or past the end of an allocation is undefined behavior — it won't segfault reliably, it will just silently produce garbage or corrupt memory.

```c
int data[5] = {10, 20, 30, 40, 50};
int *p = data;

// WRONG: reading before the array
printf("%d\n", *(p - 1));  // undefined behavior

// WRONG: reading past the end
p = &data[4];
printf("%d\n", *(p + 1));  // undefined behavior
```

The rule: `*(p + k)` is valid only if `p + k` points to an element within the allocated block (or one-past-the-end for comparison, but never for dereferencing).

A safe pattern for neighbor access:

```c
for (int *p = data; p < data + n; p++) {
    int has_prev = (p > data);
    int has_next = (p < data + n - 1);

    if (has_prev) printf("prev: %d  ", *(p - 1));
    printf("curr: %d  ", *p);
    if (has_next) printf("next: %d", *(p + 1));
    printf("\n");
}
```

### Pointer Subtraction

You can subtract two pointers that point into the same array. The result is the number of elements between them, not the number of bytes:

```c
int arr[10] = {0};
int *start = &arr[2];
int *end   = &arr[7];

ptrdiff_t distance = end - start;  // 5 (not 20 bytes)
printf("elements between: %td\n", distance);
```

`ptrdiff_t` (from `<stddef.h>`) is the signed integer type for pointer differences. Use `%td` to print it.

This is useful for computing how far into a buffer you've progressed:

```c
void process_buffer(uint8_t *buf, size_t len) {
    uint8_t *end = buf + len;
    uint8_t *p = buf;

    while (p < end) {
        // ... process *p ...
        ptrdiff_t offset = p - buf;
        printf("processed %td of %zu bytes\n", offset, len);
        p++;
    }
}
```

### Delta Encoding: A Real Compression Technique

Delta encoding stores the *difference* between consecutive values instead of the values themselves. When values change slowly (timestamps, sensor readings, stock prices), the deltas are small and compress well.

Original data: `[1000, 1002, 1001, 1005, 1003]`
Delta encoded: `[1000, +2, -1, +4, -2]`

The first value is stored as-is (the "base"). Every subsequent value is stored as the difference from its predecessor.

Why this helps with network transfer: small integers need fewer bits. If your values are 32-bit integers but the deltas fit in 8 bits, you've compressed by 4x. Protocols like Protocol Buffers use variable-length encoding where small numbers take fewer bytes — delta encoding feeds directly into that.

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>

// Encode: compute deltas in-place using pointer arithmetic
// input:  [1000, 1002, 1001, 1005, 1003]
// output: [1000,    2,   -1,    4,   -2]
void delta_encode(int32_t *data, size_t n) {
    if (n < 2) return;

    // Walk backward to avoid overwriting values we still need
    int32_t *p = data + n - 1;
    while (p > data) {
        *p = *p - *(p - 1);
        p--;
    }
    // data[0] stays as the base value
}

// Decode: reconstruct original values by accumulating deltas
// input:  [1000,    2,   -1,    4,   -2]
// output: [1000, 1002, 1001, 1005, 1003]
void delta_decode(int32_t *data, size_t n) {
    int32_t *p = data + 1;
    while (p < data + n) {
        *p = *p + *(p - 1);
        p++;
    }
}

void print_array(const char *label, int32_t *data, size_t n) {
    printf("%s: [", label);
    for (int32_t *p = data; p < data + n; p++) {
        if (p > data) printf(", ");
        printf("%d", *p);
    }
    printf("]\n");
}

int main(void) {
    int32_t data[] = {1000, 1002, 1001, 1005, 1003, 1010, 1008};
    size_t n = sizeof(data) / sizeof(data[0]);

    print_array("original", data, n);

    delta_encode(data, n);
    print_array("encoded ", data, n);

    delta_decode(data, n);
    print_array("decoded ", data, n);

    return 0;
}
```

Notice the encode function walks **backward** — from the last element toward the first. This is critical. If you walked forward and replaced `data[1]` with `data[1] - data[0]`, you'd need the original `data[1]` to compute `data[2] - data[1]`, but you've already overwritten it.

### Forward vs. Backward Traversal

The direction you walk matters when you're modifying the data you're traversing:

```c
// WRONG: forward pass destroys values needed later
void delta_encode_broken(int32_t *data, size_t n) {
    for (int32_t *p = data + 1; p < data + n; p++) {
        *p = *p - *(p - 1);  // *(p-1) was already modified!
    }
}
```

Run this on `[100, 102, 105]`:
- Step 1: `data[1] = 102 - 100 = 2` -> `[100, 2, 105]`
- Step 2: `data[2] = 105 - 2 = 103` -> `[100, 2, 103]` (wrong! should be 3)

The backward pass avoids this because each element is replaced before it's needed by its predecessor.

If you need to encode forward (maybe you're streaming and don't have the whole array), save the previous value explicitly:

```c
void delta_encode_streaming(int32_t *data, size_t n) {
    int32_t prev = data[0];
    for (int32_t *p = data + 1; p < data + n; p++) {
        int32_t original = *p;
        *p = *p - prev;
        prev = original;
    }
}
```

### Sliding Window with Pointers

Compression algorithms often look at a window of nearby values. Here's a simple moving average using three pointers:

```c
// Compute 3-element moving average, store results in output
void moving_average_3(const float *input, float *output, size_t n) {
    if (n < 3) return;

    // First and last elements: just copy (no full window)
    output[0] = input[0];
    output[n - 1] = input[n - 1];

    // Middle elements: average of previous, current, next
    const float *p = input + 1;
    float *out = output + 1;

    while (p < input + n - 1) {
        *out = (*(p - 1) + *p + *(p + 1)) / 3.0f;
        p++;
        out++;
    }
}
```

### Pointers to Different Types: Byte-Level Access

When working with network data or file formats, you often need to walk memory byte-by-byte even if the logical data is larger:

```c
#include <stdint.h>

// Print the raw bytes of an integer (little-endian on x86)
void print_bytes(void *ptr, size_t n) {
    uint8_t *byte = (uint8_t *)ptr;
    for (uint8_t *p = byte; p < byte + n; p++) {
        printf("%02x ", *p);
    }
    printf("\n");
}

int main(void) {
    int32_t value = 0x01020304;
    print_bytes(&value, sizeof(value));
    // On x86 (little-endian): 04 03 02 01
    return 0;
}
```

Casting to `uint8_t *` lets you navigate memory one byte at a time, regardless of the original type. This is how serialization libraries and network code pack data tightly for transfer.

## Exercises

### Exercise 1: Delta Encode/Decode

Compile and run the delta encoding example above. Then modify it:

1. Add a function `delta_encode_forward` that walks forward and correctly encodes (hint: save the previous original value in a local variable).
2. Verify that decoding the forward-encoded data produces the original.
3. Test with an array of timestamps: `{1682000000, 1682000060, 1682000120, 1682000180}` (one-minute intervals). What do the deltas look like?

### Exercise 2: Compression Ratio

Write a program that:

1. Generates an array of 1000 "sensor readings" — start at 5000 and add a random value between -5 and +5 at each step.
2. Delta encodes the array.
3. Counts how many deltas fit in a single byte (-128 to 127) vs. requiring a full `int32_t`.
4. Prints the theoretical compression ratio if single-byte deltas used 1 byte and the rest used 4 bytes, compared to storing all 1000 values as `int32_t`.

### Exercise 3: Double Delta Encoding

Some data (like linearly increasing timestamps) has constant deltas. Delta encoding those gives all identical values — but double-delta encoding (delta of deltas) gives mostly zeros, which compress even better.

Implement `double_delta_encode` and `double_delta_decode`:

```c
// Example:
// original:      [1000, 1010, 1020, 1030, 1025, 1035]
// first delta:   [1000,   10,   10,   10,   -5,   10]
// double delta:  [1000,   10,    0,    0,  -15,   15]
```

Use pointer arithmetic for all traversals. Verify round-trip correctness: `double_delta_decode(double_delta_encode(data))` must produce the original data.

### Exercise 4: Run-Length Encoding with Pointers

Implement run-length encoding (RLE) using pointer-based traversal:

```c
// Input:  [5, 5, 5, 3, 3, 8, 8, 8, 8, 1]
// Output: [(5,3), (3,2), (8,4), (1,1)]  — stored as [5, 3, 3, 2, 8, 4, 1, 1]

size_t rle_encode(const int *input, size_t n, int *output);
void   rle_decode(const int *input, size_t n, int *output);
```

Use a `curr` pointer and a `run_start` pointer. Advance `curr` while `*curr == *run_start`, then emit the value and the count (`curr - run_start`). This naturally uses pointer subtraction to compute run lengths.

### Exercise 5: XOR Delta for Network Transfer

A variant used in some network protocols: instead of subtracting previous values, XOR them. XOR deltas are useful when data isn't numerically sequential but has many shared bits:

```c
void xor_encode(uint32_t *data, size_t n);
void xor_decode(uint32_t *data, size_t n);
```

1. Implement both using pointer arithmetic.
2. XOR encoding is its own inverse — `xor_encode` and `xor_decode` have the same structure as delta encoding/decoding. Why? (Hint: think about what `a ^ b ^ a` equals.)
3. Test with an array of IPv4 addresses stored as `uint32_t`. Addresses in the same subnet share upper bits — what do the XOR deltas look like?

## What Comes Next

The pointer arithmetic techniques here — neighbor access, sliding windows, multi-pass traversal — appear everywhere in systems programming. Unit 09 uses similar patterns for file I/O buffers, and Unit 15 (Networking) is where you'll pack and unpack data for actual network transfer. Delta encoding is also a building block for more sophisticated compression (LZ77, zstd) that you may encounter when working with storage or network protocols.
