# Unit 08: Data Structures in C

## Overview

Implementing data structures from scratch in C is the best way to solidify pointer mastery. In higher-level languages, data structures are library imports. In C, you build them yourself — and that forces you to think precisely about memory layout, ownership, and performance. This unit covers the structures every systems programmer needs: linked lists, dynamic arrays, hash tables, binary search trees, and memory arenas. By the end, returning to a list in Python or a map in Go will feel like magic you now understand.

## Prerequisites

- Unit 07 (Pointers & Memory Management) — malloc/free, pointer arithmetic, dynamic arrays
- Unit 05 (C Language Fundamentals) — types, arrays, functions
- Unit 06 (Compilation & Build Tools) — multi-file project layout

## Learning Objectives

- Implement singly and doubly linked lists with correct memory management
- Build a generic dynamic array (growable buffer)
- Implement a hash table with open addressing or chaining
- Implement a binary search tree with insert, search, and delete
- Use memory arenas for fast allocation in performance-sensitive code
- Choose the right data structure for a given problem based on access patterns

## Reading / Resources

- K&R — Chapter 6 (Structures)
- `man 3 qsort`, `man 3 bsearch` — standard library searching and sorting
- The Practice of Programming by Kernighan & Pike — Chapter 2

## Concepts

### Structs and Self-Referential Types

```c
// A node in a singly linked list
typedef struct Node {
    int value;
    struct Node *next;  // must use 'struct Node', not 'Node' — typedef not yet complete
} Node;

// Allocate a node
Node *node_new(int value) {
    Node *n = malloc(sizeof(Node));
    if (!n) return NULL;
    n->value = value;
    n->next = NULL;
    return n;
}
```

### Singly Linked List

```c
typedef struct {
    Node *head;
    size_t len;
} List;

void list_push_front(List *l, int value) {
    Node *n = node_new(value);
    n->next = l->head;
    l->head = n;
    l->len++;
}

void list_push_back(List *l, int value) {
    Node *n = node_new(value);
    if (!l->head) { l->head = n; }
    else {
        Node *cur = l->head;
        while (cur->next) cur = cur->next;
        cur->next = n;
    }
    l->len++;
}

int list_pop_front(List *l, int *out) {
    if (!l->head) return 0;
    Node *n = l->head;
    l->head = n->next;
    *out = n->value;
    free(n);
    l->len--;
    return 1;
}

void list_free(List *l) {
    Node *cur = l->head;
    while (cur) {
        Node *next = cur->next;
        free(cur);
        cur = next;
    }
    l->head = NULL;
    l->len = 0;
}
```

O(1) push_front, O(n) push_back. To get O(1) push_back, keep a `tail` pointer.

### Doubly Linked List

```c
typedef struct DNode {
    int value;
    struct DNode *prev;
    struct DNode *next;
} DNode;

typedef struct {
    DNode *head;
    DNode *tail;
    size_t len;
} DList;

// O(1) insert at either end, O(1) remove given a pointer to the node
void dlist_push_back(DList *l, int value) {
    DNode *n = malloc(sizeof(DNode));
    n->value = value;
    n->next = NULL;
    n->prev = l->tail;
    if (l->tail) l->tail->next = n;
    else l->head = n;
    l->tail = n;
    l->len++;
}

void dlist_remove(DList *l, DNode *n) {
    if (n->prev) n->prev->next = n->next;
    else l->head = n->next;
    if (n->next) n->next->prev = n->prev;
    else l->tail = n->prev;
    free(n);
    l->len--;
}
```

### Dynamic Array (Vector)

A contiguous array that grows automatically. This is the default data structure for most problems — cache-friendly, random access O(1), amortized O(1) append.

```c
typedef struct {
    int    *data;
    size_t  len;
    size_t  cap;
} Vec;

void vec_init(Vec *v) {
    v->data = NULL;
    v->len = v->cap = 0;
}

void vec_push(Vec *v, int value) {
    if (v->len == v->cap) {
        size_t new_cap = v->cap ? v->cap * 2 : 8;
        int *new_data = realloc(v->data, new_cap * sizeof(int));
        if (!new_data) { perror("realloc"); exit(1); }
        v->data = new_data;
        v->cap = new_cap;
    }
    v->data[v->len++] = value;
}

void vec_free(Vec *v) {
    free(v->data);
    vec_init(v);
}
```

The doubling strategy gives amortized O(1) push — total copies across all pushes = O(n).

### Hash Table

A hash table maps keys to values with average O(1) insert, lookup, and delete.

**Chaining** (each bucket is a linked list of entries with the same hash):

```c
#define HT_SIZE 64

typedef struct HEntry {
    char        *key;    // owned copy
    int          value;
    struct HEntry *next;
} HEntry;

typedef struct { HEntry *buckets[HT_SIZE]; } HTable;

static size_t hash(const char *key) {
    // djb2 XOR variant
    size_t h = 5381;
    for (; *key; key++) h = h * 33 ^ (unsigned char)*key;
    return h % HT_SIZE;
}

void ht_set(HTable *ht, const char *key, int value) {
    size_t idx = hash(key);
    for (HEntry *e = ht->buckets[idx]; e; e = e->next) {
        if (strcmp(e->key, key) == 0) { e->value = value; return; }
    }
    HEntry *e = malloc(sizeof(HEntry));
    e->key = strdup(key);
    e->value = value;
    e->next = ht->buckets[idx];
    ht->buckets[idx] = e;
}

int ht_get(const HTable *ht, const char *key, int *out) {
    size_t idx = hash(key);
    for (HEntry *e = ht->buckets[idx]; e; e = e->next) {
        if (strcmp(e->key, key) == 0) { *out = e->value; return 1; }
    }
    return 0;
}

void ht_free(HTable *ht) {
    for (int i = 0; i < HT_SIZE; i++) {
        HEntry *e = ht->buckets[i];
        while (e) { HEntry *next = e->next; free(e->key); free(e); e = next; }
        ht->buckets[i] = NULL;
    }
}
```

Real hash tables also need: resize when load factor exceeds ~0.7, better hash functions (SipHash, FNV-1a), open addressing as an alternative to chaining.

### Binary Search Tree

```c
typedef struct BST {
    int val;
    struct BST *left;
    struct BST *right;
} BST;

BST *bst_insert(BST *root, int val) {
    if (!root) {
        BST *n = malloc(sizeof(BST));
        n->val = val; n->left = n->right = NULL;
        return n;
    }
    if (val < root->val) root->left  = bst_insert(root->left,  val);
    else if (val > root->val) root->right = bst_insert(root->right, val);
    return root;
}

int bst_contains(BST *root, int val) {
    if (!root) return 0;
    if (val == root->val) return 1;
    return val < root->val
        ? bst_contains(root->left,  val)
        : bst_contains(root->right, val);
}

// In-order traversal: visits values in sorted order
void bst_inorder(BST *root, void (*visit)(int)) {
    if (!root) return;
    bst_inorder(root->left, visit);
    visit(root->val);
    bst_inorder(root->right, visit);
}

void bst_free(BST *root) {
    if (!root) return;
    bst_free(root->left);
    bst_free(root->right);
    free(root);
}
```

A plain BST degenerates to O(n) in the worst case (sorted input). Production systems use self-balancing trees (AVL, red-black) or skip lists. The Linux kernel uses red-black trees extensively (see `<linux/rbtree.h>`).

### Memory Arenas

An arena allocator allocates from a large pre-allocated block. Individual frees are O(1) (no-op); the entire arena is freed at once. Used for request/response cycles, AST nodes, parser state — anything with a clear lifetime.

```c
typedef struct {
    char  *buf;
    size_t used;
    size_t cap;
} Arena;

Arena *arena_new(size_t cap) {
    Arena *a = malloc(sizeof(Arena));
    a->buf = malloc(cap);
    a->used = 0;
    a->cap = cap;
    return a;
}

void *arena_alloc(Arena *a, size_t size) {
    // align to 8 bytes
    size = (size + 7) & ~(size_t)7;
    if (a->used + size > a->cap) return NULL;  // or grow
    void *ptr = a->buf + a->used;
    a->used += size;
    return ptr;
}

void arena_reset(Arena *a) { a->used = 0; }  // "free" everything at once

void arena_free(Arena *a) {
    free(a->buf);
    free(a);
}
```

Arena allocation is O(1) and has zero fragmentation. The entire HTTP request server example from Unit 15 could allocate all per-request strings from an arena and reset it after sending the response.

### Choosing a Data Structure

| Structure | Access | Insert | Delete | Best for |
|-----------|--------|--------|--------|----------|
| Array | O(1) | O(n) | O(n) | Indexed access, cache performance |
| Dynamic array | O(1) | O(1)* | O(n) | Default choice, sequential access |
| Linked list | O(n) | O(1) | O(1)† | Frequent insert/remove in middle |
| Hash table | O(1)* | O(1)* | O(1)* | Key-value lookup |
| BST | O(log n)* | O(log n)* | O(log n)* | Sorted iteration + lookup |

*amortized / average case  †given a pointer to the node

In practice, prefer dynamic arrays. They are cache-friendly (Drepper Unit 01 applies here) and simpler. Reach for hash tables when you need key lookup. Linked lists are less useful than they appear — pointer-chasing is cache-hostile.

## Exercises

### Exercise 1: Doubly Linked List

Implement a complete doubly linked list with:
```c
DList *dlist_new(void);
void dlist_push_front(DList *l, int val);
void dlist_push_back(DList *l, int val);
int  dlist_pop_front(DList *l, int *out);
int  dlist_pop_back(DList *l, int *out);
void dlist_remove_value(DList *l, int val);
void dlist_reverse(DList *l);
void dlist_free(DList *l);
```

Write a test that: pushes 1000 values, reverses the list, verifies the order, removes all even values, and checks the final count. Run under valgrind — zero leaks.

### Exercise 2: Generic Dynamic Array

Make a type-generic dynamic array using `void *` and a user-supplied element size:

```c
typedef struct {
    void  *data;
    size_t len;
    size_t cap;
    size_t elem_size;
} GenVec;

GenVec *gvec_new(size_t elem_size);
void    gvec_push(GenVec *v, const void *elem);
void   *gvec_get(GenVec *v, size_t idx);
void    gvec_sort(GenVec *v, int (*cmp)(const void *, const void *));
void    gvec_free(GenVec *v);
```

Test it with both `int` and a `struct Point { int x, y; }`.

### Exercise 3: Hash Table with Resize

Extend the hash table from the Concepts section to:
1. Automatically resize (double buckets) when load factor exceeds 0.7
2. Support deletion (`ht_delete`)
3. Support iteration (`ht_foreach(ht, callback)`)

Test with 10,000 insertions. Verify no keys are lost after resize. Run under valgrind.

### Exercise 4: Binary Search Tree

Implement a full BST supporting:
```c
BST *bst_insert(BST *root, int val);
int  bst_contains(BST *root, int val);
BST *bst_delete(BST *root, int val);   // the hard one
int  bst_height(BST *root);
void bst_inorder(BST *root, void (*visit)(int));
void bst_free(BST *root);
```

Deletion requires three cases: leaf node, one child, two children (replace with in-order successor). Test by inserting 100 random values, deleting half, and verifying the remaining values are correct via in-order traversal.

### Exercise 5: Word Frequency Counter

Using your hash table, write a program that:
1. Reads a text file word by word
2. Counts the frequency of each word (case-insensitive)
3. Prints the top 20 most frequent words with their counts, sorted descending

Test with a large text file (e.g. download a Project Gutenberg book):
```bash
curl -o book.txt https://www.gutenberg.org/files/1342/1342-0.txt
./wordfreq book.txt
```

### Exercise 6: Arena Allocator Benchmark

Compare allocation performance:
1. `malloc`/`free` per node for a linked list of 1,000,000 nodes
2. Arena allocation for the same list (allocate all nodes from one arena)
3. `free` all at once vs. individual `free` calls

Measure with `time` and compare. Observe the difference in fragmentation and total allocator overhead.

## What Comes Next

Unit 09 covers filesystem and file I/O — the POSIX API for reading and writing files. The data structures you have built here (especially dynamic arrays and hash tables) will appear throughout later units as you build more complex programs.
