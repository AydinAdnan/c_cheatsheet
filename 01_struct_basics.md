# Chapter 1: Struct Fundamentals

## 1.1 What is a Struct?

A `struct` (structure) is a **user-defined composite data type** that groups related variables of
different types under a single name. Each variable inside a struct is called a **member** (or field).

**Why structs exist:** C has only primitive types (`int`, `char`, `float`, etc.). Real-world data
(a sensor reading, a network packet, an employee record) requires grouping multiple values together.
Structs solve this.

```c
// Without structs — scattered, error-prone
int sensor_id;
float temperature;
float humidity;
uint32_t timestamp;

// With structs — organized, portable
struct SensorReading {
    int sensor_id;
    float temperature;
    float humidity;
    uint32_t timestamp;
};
```

---

## 1.2 Declaration Syntax

### Basic Declaration
```c
struct TagName {
    type1 member1;
    type2 member2;
    // ...
};  // ← Semicolon is MANDATORY
```

⚠️ **Edge Case — Forgetting the semicolon:**
```c
struct Point {
    int x;
    int y;
}  // Missing semicolon!

int main(void) { ... }
// Compiler error: often cryptic, may say "expected ';' before 'int'"
```

### Declaration + Variable Definition (simultaneously)
```c
struct Point {
    int x;
    int y;
} p1, p2;  // p1 and p2 are variables of type 'struct Point'
```

### Anonymous Struct (no tag name)
```c
struct {
    int x;
    int y;
} origin;  // Can only use 'origin'; cannot create new variables of this type later

// origin.x = 0;  ✅ Works
// struct ??? another; — impossible, no type name exists
```

💡 **When to use anonymous structs:** Almost never. They are sometimes used inside unions
(covered in Chapter 5) or for one-off configurations in embedded firmware.

---

## 1.3 The `typedef` Pattern

`typedef` creates an alias for a type, eliminating the need to write `struct` everywhere.

```c
// Pattern 1: Separate typedef
struct Point {
    int x;
    int y;
};
typedef struct Point Point;

// Pattern 2: Combined (most common in practice)
typedef struct {
    int x;
    int y;
} Point;

// Pattern 3: Self-referencing (MUST keep the tag)
typedef struct Node {
    int data;
    struct Node *next;  // Must use 'struct Node', not 'Node' (not yet defined)
} Node;
```

⚠️ **Edge Case — Self-referencing without tag:**
```c
typedef struct {          // No tag!
    int data;
    Node *next;           // ERROR: 'Node' is not defined yet at this point
} Node;
```
The typedef name `Node` is only available AFTER the closing `}`. Inside the struct body,
you must use `struct TagName`.

### 🔧 Embedded Convention (Linux Kernel Style)
The Linux kernel avoids `typedef` for structs to make the `struct` keyword explicit,
improving readability in large codebases:
```c
struct i2c_msg {
    __u16 addr;
    __u16 flags;
    __u16 len;
    __u8 *buf;
};
// Always referred to as 'struct i2c_msg', never just 'i2c_msg'
```

---

## 1.4 Creating Variables (Instantiation)

```c
typedef struct {
    int x;
    int y;
} Point;

// Method 1: Default (uninitialized on stack — contains garbage!)
Point p1;
// p1.x is INDETERMINATE — reading it is Undefined Behavior (UB)

// Method 2: Zero-initialize
Point p2 = {0};  // All members set to 0

// Method 3: Designated initializers (C99+) — PREFERRED
Point p3 = {.x = 10, .y = 20};

// Method 4: Positional initializers (fragile — breaks if member order changes)
Point p4 = {10, 20};  // x=10, y=20 based on declaration order

// Method 5: Compound literal (C99+) — for temporary/inline use
Point p5 = (Point){.x = 5, .y = 15};
```

⚠️ **Edge Case — Partial initialization:**
```c
typedef struct {
    int a, b, c, d;
} Quad;

Quad q = {.a = 1, .c = 3};
// q.a = 1, q.b = 0, q.c = 3, q.d = 0
// Members NOT explicitly initialized are zero-initialized (by the C standard)
```

⚠️ **Edge Case — Uninitialized stack struct:**
```c
void foo(void) {
    Point p;       // On stack, NOT initialized
    printf("%d", p.x);  // UB! May print garbage, may crash, may "work"
}
```

💡 **Best Practice:** Always initialize structs. Use `= {0}` or designated initializers.

---

## 1.5 Accessing Members

### Dot Operator (`.`) — Direct access
```c
Point p = {.x = 10, .y = 20};
printf("x=%d, y=%d\n", p.x, p.y);

p.x = 100;  // Modify
```

### Arrow Operator (`->`) — Pointer access
```c
Point p = {.x = 10, .y = 20};
Point *ptr = &p;

printf("x=%d\n", ptr->x);     // Equivalent to (*ptr).x
ptr->y = 99;                   // Modify through pointer
```

💡 **Why `->` exists:**
`(*ptr).x` is awkward and error-prone (the parentheses are mandatory due to operator precedence).
`ptr->x` is syntactic sugar that is clearer and universally used.

⚠️ **Edge Case — Operator precedence trap:**
```c
*ptr.x    // WRONG! Parsed as *(ptr.x) — 'ptr' is a pointer, not a struct
(*ptr).x  // Correct
ptr->x    // Correct and preferred
```

---

## 1.6 Struct Assignment (Copy Semantics)

Structs in C support **direct assignment** — the entire struct is copied member-by-member:

```c
Point a = {.x = 1, .y = 2};
Point b = a;    // Deep copy of all members
b.x = 99;      // Does NOT affect 'a'
printf("%d\n", a.x);  // Still 1
```

### ⚠️ Shallow Copy Problem (Pointer Members)
```c
typedef struct {
    char *name;  // Pointer, not array!
    int age;
} Person;

Person a = {.name = "Alice", .age = 30};
Person b = a;  // Copies the POINTER, not the string

// Both a.name and b.name point to the SAME string literal
// If name were malloc'd, modifying through b.name would affect a.name too!
```

```c
// Safe deep copy for dynamically allocated members:
Person deep_copy(const Person *src) {
    Person dst;
    dst.age = src->age;
    dst.name = malloc(strlen(src->name) + 1);
    if (dst.name) strcpy(dst.name, src->name);
    return dst;
}
```

---

## 1.7 Struct Comparison

⚠️ **You CANNOT use `==` to compare structs:**
```c
Point a = {1, 2};
Point b = {1, 2};
if (a == b) { ... }  // COMPILE ERROR!
```

**Why?** Padding bytes (see Chapter 3) may contain garbage. `memcmp` would compare padding too.

### Correct Ways to Compare:
```c
// Method 1: Member-by-member (safest, most explicit)
bool points_equal(const Point *a, const Point *b) {
    return a->x == b->x && a->y == b->y;
}

// Method 2: memcmp (ONLY safe if struct has NO padding and NO pointer members)
// Generally AVOID this — padding makes it unreliable
if (memcmp(&a, &b, sizeof(Point)) == 0) { ... }  // ⚠️ Risky!
```

💡 **Embedded Tip:** If you must use `memcmp` (e.g., comparing packet buffers), zero-initialize
the struct first with `memset(&s, 0, sizeof(s))` to clear padding bytes.

---

## 1.8 Passing Structs to Functions

### By Value (copy)
```c
void print_point(Point p) {      // Entire struct is COPIED onto the stack
    printf("(%d, %d)\n", p.x, p.y);
    p.x = 999;                   // Modifies the LOCAL copy only
}

Point p = {1, 2};
print_point(p);
printf("%d\n", p.x);  // Still 1 — original unchanged
```

### By Pointer (reference — preferred for large structs)
```c
void move_point(Point *p, int dx, int dy) {
    p->x += dx;
    p->y += dy;
}

Point p = {1, 2};
move_point(&p, 10, 20);
printf("(%d, %d)\n", p.x, p.y);  // (11, 22)
```

### By `const` Pointer (read-only reference — safest)
```c
void print_point(const Point *p) {
    printf("(%d, %d)\n", p->x, p->y);
    // p->x = 5;  // COMPILE ERROR — const prevents modification
}
```

💡 **Rule of Thumb:**
- Small structs (≤ 16 bytes): pass by value is fine (often faster, avoids indirection)
- Large structs: ALWAYS pass by `const` pointer (avoids expensive stack copies)
- When you need to modify: pass by non-const pointer

### 🔧 Embedded Consideration
On ARM Cortex-M, small structs (≤ 4 words / 16 bytes) may be passed in registers (R0-R3),
making pass-by-value efficient. Larger structs are pushed onto the stack, which is wasteful
on memory-constrained MCUs with 2-64 KB RAM.

---

## 1.9 Returning Structs from Functions

```c
Point make_point(int x, int y) {
    return (Point){.x = x, .y = y};  // Compound literal returned by value
}

Point p = make_point(10, 20);
```

⚠️ **Edge Case — DO NOT return pointers to local structs:**
```c
Point *bad_make_point(int x, int y) {
    Point p = {.x = x, .y = y};  // Local variable on stack
    return &p;  // DANGLING POINTER! Stack frame destroyed after return
}
// Using the returned pointer is Undefined Behavior
```

💡 **Fix:** Either return by value, use `malloc`, or have the caller provide a buffer:
```c
// Option 1: malloc (caller must free)
Point *make_point_heap(int x, int y) {
    Point *p = malloc(sizeof(Point));
    if (p) { p->x = x; p->y = y; }
    return p;
}

// Option 2: Output parameter (no allocation, very common in embedded)
void make_point_out(int x, int y, Point *out) {
    out->x = x;
    out->y = y;
}
```

---

## 1.10 Struct Size and `sizeof`

```c
typedef struct {
    char a;     // 1 byte
    int b;      // 4 bytes
    char c;     // 1 byte
} Example;

printf("Size: %zu\n", sizeof(Example));  // Likely 12, NOT 6!
```

**Why 12?** Due to **alignment and padding** (covered extensively in Chapter 3).
The compiler inserts invisible padding bytes to align members to their natural boundaries.

---

## 1.11 Flexible Array Members (C99+)

A struct's **last member** can be an incomplete array (no size specified):

```c
typedef struct {
    uint16_t length;
    uint8_t data[];  // Flexible array member — size determined at runtime
} Packet;

// Allocation:
Packet *pkt = malloc(sizeof(Packet) + 100);  // 100 bytes for data[]
pkt->length = 100;
pkt->data[0] = 0xFF;

// sizeof(Packet) does NOT include data[] — only the fixed members
printf("%zu\n", sizeof(Packet));  // 2 (just the length field + possible padding)
```

### 🔧 Embedded Use Case — Variable-Length Protocol Messages
```c
typedef struct {
    uint8_t  msg_id;
    uint8_t  flags;
    uint16_t payload_len;
    uint8_t  payload[];  // FAM — maps directly onto received buffer
} __attribute__((packed)) ProtocolMsg;

// Cast a received DMA buffer to this struct:
void handle_rx(uint8_t *dma_buf, size_t len) {
    ProtocolMsg *msg = (ProtocolMsg *)dma_buf;
    process(msg->msg_id, msg->payload, msg->payload_len);
}
```

⚠️ **Edge Cases:**
- You CANNOT have an array of structs with FAMs
- You CANNOT put a FAM struct on the stack (size unknown at compile time)
- The FAM must be the LAST member
- `sizeof` does not include the FAM

---

## 1.12 Complete Example: Sensor Data Logger

```c
#include <stdio.h>
#include <stdint.h>
#include <time.h>

typedef enum {
    SENSOR_TEMP,
    SENSOR_HUMIDITY,
    SENSOR_PRESSURE
} SensorType;

typedef struct {
    uint8_t     sensor_id;
    SensorType  type;
    float       value;
    uint32_t    timestamp;
} SensorReading;

void print_reading(const SensorReading *r) {
    const char *type_str[] = {"Temperature", "Humidity", "Pressure"};
    printf("[%u] Sensor #%u (%s): %.2f\n",
           r->timestamp, r->sensor_id, type_str[r->type], r->value);
}

int main(void) {
    // Array of struct readings
    SensorReading log[] = {
        {.sensor_id = 1, .type = SENSOR_TEMP,     .value = 23.5f, .timestamp = 1000},
        {.sensor_id = 1, .type = SENSOR_HUMIDITY,  .value = 65.0f, .timestamp = 1001},
        {.sensor_id = 2, .type = SENSOR_PRESSURE,  .value = 1013.25f, .timestamp = 1002},
    };

    size_t count = sizeof(log) / sizeof(log[0]);
    for (size_t i = 0; i < count; i++) {
        print_reading(&log[i]);
    }

    return 0;
}
```

**Output:**
```
[1000] Sensor #1 (Temperature): 23.50
[1001] Sensor #1 (Humidity): 65.00
[1002] Sensor #2 (Pressure): 1013.25
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| Declaration | `struct Tag { ... };` — semicolon required |
| typedef | Avoids writing `struct` everywhere; self-ref needs tag |
| Initialization | Use designated initializers `{.x = val}` (C99+) |
| Access | `.` for direct, `->` for pointer |
| Assignment | Memberwise copy; shallow for pointer members |
| Comparison | No `==`; compare member-by-member |
| Passing | By value (small), const pointer (large/read), pointer (modify) |
| FAM | Last member `type arr[];` — runtime-sized |

---

*Next: [Chapter 2 — Advanced Struct Techniques →](02_struct_advanced.md)*
