# Unit 13: Threads & Synchronization

## Overview

Threads are like processes that share memory. Multiple threads within a process run concurrently and see the same global variables, heap, and file descriptors — which makes them fast to create and communicate through, but also dangerous: unsynchronized access to shared data produces **race conditions** that are notoriously hard to reproduce and debug. This unit covers POSIX threads (pthreads), the synchronization primitives that make concurrent programming safe, and the common patterns and pitfalls of multithreaded code.

## Prerequisites

- Unit 11 (Processes & Signals) — process model, fork/exec
- Unit 07 (Pointers & Memory Management) — pointer and heap comfort

## Learning Objectives

- Create, join, and detach threads with the pthreads API
- Use mutexes to protect shared data
- Use condition variables for thread coordination
- Recognize race conditions and deadlocks
- Implement thread-safe data structures
- Understand thread-local storage
- Use atomic operations from `<stdatomic.h>` for simple counters

## Reading / Resources

- `man 7 pthreads` — pthreads overview
- `man 3 pthread_create`, `man 3 pthread_mutex_lock`, `man 3 pthread_cond_wait`
- `man 7 memory_model` — C11 memory model
- The Art of Multiprocessor Programming by Herlihy & Shavit (reference)

## Concepts

### Creating Threads

```c
#include <pthread.h>

void *worker(void *arg) {
    int id = *(int *)arg;
    printf("Thread %d running\n", id);
    return NULL;  // or return a value pointer
}

int main(void) {
    pthread_t tid;
    int id = 42;

    if (pthread_create(&tid, NULL, worker, &id) != 0) {
        perror("pthread_create");
        exit(1);
    }

    pthread_join(tid, NULL);  // wait for thread to finish
    return 0;
}
```

Compile with `-lpthread`:
```bash
gcc -Wall -g -o program program.c -lpthread
```

### Thread Lifecycle

- **Create**: `pthread_create` starts a new thread executing the given function
- **Join**: `pthread_join(tid, &retval)` blocks until the thread exits, collects return value
- **Detach**: `pthread_detach(tid)` marks thread so its resources are freed automatically on exit (no join needed)
- **Exit**: Thread function returns, or `pthread_exit(retval)` is called explicitly

A thread that is neither joined nor detached leaks resources (similar to a zombie process).

### Race Conditions

A race condition occurs when the result of a program depends on the interleaving of thread executions:

```c
// BROKEN: race condition on `counter`
int counter = 0;

void *increment(void *arg) {
    for (int i = 0; i < 1000000; i++) {
        counter++;  // NOT atomic: read, increment, write — three operations
    }
    return NULL;
}

// Run two threads: expected result 2000000, actual result varies
```

On x86, `counter++` compiles to at least three instructions. Two threads can both read the same value, both increment locally, and both write back — losing one increment.

### Mutexes

A **mutex** (mutual exclusion lock) ensures only one thread executes a critical section at a time:

```c
#include <pthread.h>

pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
int counter = 0;

void *increment(void *arg) {
    for (int i = 0; i < 1000000; i++) {
        pthread_mutex_lock(&lock);
        counter++;  // protected — only one thread here at a time
        pthread_mutex_unlock(&lock);
    }
    return NULL;
}
```

Rules:
- Always lock before accessing shared data, always unlock after
- Keep critical sections short
- Never return or `pthread_exit` while holding a lock
- Never call a function that might block while holding a lock

Dynamic initialization:
```c
pthread_mutex_t lock;
pthread_mutex_init(&lock, NULL);
// ... use ...
pthread_mutex_destroy(&lock);
```

### Deadlock

Deadlock occurs when two threads each hold a lock the other wants:

```c
// Thread 1: lock(A), lock(B)
// Thread 2: lock(B), lock(A)  <- deadlock!
```

Prevention: always acquire multiple locks in the same order across all threads.

### Condition Variables

A **condition variable** lets a thread sleep until a condition becomes true, without busy-waiting:

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t  cond = PTHREAD_COND_INITIALIZER;
int ready = 0;

void *producer(void *arg) {
    // ... produce something ...
    pthread_mutex_lock(&lock);
    ready = 1;
    pthread_cond_signal(&cond);  // wake one waiting thread
    pthread_mutex_unlock(&lock);
    return NULL;
}

void *consumer(void *arg) {
    pthread_mutex_lock(&lock);
    while (!ready) {  // loop to handle spurious wakeups
        pthread_cond_wait(&cond, &lock);
        // wait atomically releases lock and sleeps
        // when woken, re-acquires lock before returning
    }
    // consume
    pthread_mutex_unlock(&lock);
    return NULL;
}
```

**Always check the condition in a `while` loop** — condition variables can have spurious wakeups.

`pthread_cond_broadcast` wakes all waiting threads (vs `signal` which wakes one).

### Producer-Consumer Pattern

The classic multithreaded pattern: producers add items to a bounded queue, consumers remove them:

```c
#define QUEUE_SIZE 16

typedef struct {
    int data[QUEUE_SIZE];
    int head, tail, count;
    pthread_mutex_t lock;
    pthread_cond_t  not_full;
    pthread_cond_t  not_empty;
} Queue;

void queue_push(Queue *q, int value) {
    pthread_mutex_lock(&q->lock);
    while (q->count == QUEUE_SIZE)
        pthread_cond_wait(&q->not_full, &q->lock);
    q->data[q->tail] = value;
    q->tail = (q->tail + 1) % QUEUE_SIZE;
    q->count++;
    pthread_cond_signal(&q->not_empty);
    pthread_mutex_unlock(&q->lock);
}

int queue_pop(Queue *q) {
    pthread_mutex_lock(&q->lock);
    while (q->count == 0)
        pthread_cond_wait(&q->not_empty, &q->lock);
    int value = q->data[q->head];
    q->head = (q->head + 1) % QUEUE_SIZE;
    q->count--;
    pthread_cond_signal(&q->not_full);
    pthread_mutex_unlock(&q->lock);
    return value;
}
```

### Read-Write Locks

When data is read frequently but written rarely, a reader-writer lock allows multiple concurrent readers but exclusive writers:

```c
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;

// Multiple readers can hold this simultaneously
pthread_rwlock_rdlock(&rwlock);
// ... read shared data ...
pthread_rwlock_unlock(&rwlock);

// Only one writer, exclusive
pthread_rwlock_wrlock(&rwlock);
// ... modify shared data ...
pthread_rwlock_unlock(&rwlock);
```

### Thread-Local Storage

Variables declared `__thread` (or `_Thread_local` in C11) have a separate instance per thread:

```c
__thread int per_thread_counter = 0;

void *worker(void *arg) {
    per_thread_counter++;  // each thread has its own copy
    return NULL;
}
```

Useful for per-thread caches, error state, or any data that should not be shared.

### Atomic Operations (C11)

For simple operations like incrementing a counter, `<stdatomic.h>` avoids the overhead of a mutex:

```c
#include <stdatomic.h>

atomic_int counter = ATOMIC_VAR_INIT(0);

void *increment(void *arg) {
    for (int i = 0; i < 1000000; i++) {
        atomic_fetch_add(&counter, 1);
    }
    return NULL;
}
```

Atomics are correct for counters and flags, but do not replace mutexes for protecting compound operations (read-modify-write sequences involving multiple variables).

## Exercises

### Exercise 1: Demonstrate a Race Condition

Write a program with two threads each incrementing a shared `counter` 1,000,000 times without synchronization. Run it many times and observe that the result is often not 2,000,000.

Then fix it with a mutex. Verify the result is always correct. Compare the execution time with and without the mutex (use `time ./program`).

### Exercise 2: Producer-Consumer Queue

Implement the bounded queue from the Concepts section with:
- Thread-safe `push` and `pop`
- 3 producer threads, 2 consumer threads
- Each producer pushes 100 items (0–99)
- Each consumer pops items and prints them
- Main thread joins all threads and verifies total item count

Run under ThreadSanitizer to verify no races:
```bash
gcc -Wall -g -fsanitize=thread -o program program.c -lpthread
./program
```

### Exercise 3: Thread Pool

Implement a thread pool that:
- Creates N worker threads at startup
- Has a queue of function pointer + argument tasks
- Workers pull tasks from the queue and execute them
- Supports `pool_submit(pool, func, arg)` from the main thread
- Supports `pool_wait(pool)` to wait for all submitted tasks to complete
- Supports `pool_destroy(pool)` to shut down

Test by submitting 1000 tasks that each compute a small amount of work.

### Exercise 4: Dining Philosophers

The classic deadlock demonstration: 5 philosophers sit at a table with 5 forks. Each needs two forks to eat. Naive implementation deadlocks.

Implement it with:
1. First the naive version — demonstrate the deadlock
2. Then fix it using the resource ordering solution (always pick up lower-numbered fork first)

### Exercise 5: Cache-Friendly Parallel Matrix Multiply

Revisit the matrix multiplication from Unit 07. Add parallelism: divide the rows of the result matrix among N threads.

Measure speedup vs. the single-threaded version. Does it scale linearly with the number of threads? Why or why not? (Hint: think about the memory access patterns from Unit 01.)

## What Comes Next

Unit 14 covers the debugging and profiling tools that make sense of everything that goes wrong: gdb for step-by-step debugging, valgrind for memory errors, ThreadSanitizer for race conditions, and perf for performance analysis.
