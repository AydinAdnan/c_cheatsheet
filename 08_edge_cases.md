# Chapter 8: Edge Cases, Pitfalls & Undefined Behavior

## 8.1 Undefined Behavior (UB) with Structs

### 1. Reading Uninitialized Members
```c
typedef struct { int x, y; } Point;

Point p;                    // Stack — not initialized
printf("%d\n", p.x);       // UB! Could print anything, crash, or "work"

// Fix:
Point p = {0};              // Zero-init
Point p = {.x = 1, .y = 2}; // Designated init
```

### 2. Dereferencing NULL Struct Pointer
```c
Point *p = NULL;
p->x = 5;    // UB! Segfault on most systems, but the compiler may assume it never happens
```

### 3. Use-After-Free
```c
Point *p = malloc(sizeof(Point));
p->x = 10;
free(p);
p->x = 20;   // UB! Memory may be reused
```
💡 **Fix:** Set pointer to NULL after free: `free(p); p = NULL;`

### 4. Buffer Overflow in Struct Arrays
```c
typedef struct { char name[8]; int id; } Record;
Record r = {.id = 42};
strcpy(r.name, "VeryLongName");  // OVERFLOW! Writes past name[7], corrupts 'id'
// Use strncpy(r.name, "VeryLongName", sizeof(r.name) - 1);
```

### 5. Returning Local Struct by Pointer
```c
Point *bad(void) {
    Point local = {1, 2};
    return &local;        // DANGLING POINTER — stack frame gone
}
```

---

## 8.2 Implementation-Defined Behavior

These behave consistently on a given compiler/platform but may differ across systems:

### 1. Struct Padding and Size
```c
struct S { char a; int b; };
// sizeof(S) could be 5 (packed), 8 (common), or other values
// NEVER hardcode struct sizes — always use sizeof()
```

### 2. Bit-Field Ordering
```c
struct { uint8_t low : 4; uint8_t high : 4; } bf;
bf.low = 0xA;
bf.high = 0xB;
// Is the byte 0xBA or 0xAB? Depends on compiler and architecture!
// GCC x86/ARM: low is LSB → byte = 0xBA
```

### 3. Bit-Field Signedness
```c
struct { int val : 3; } s;  // Is 'int' signed or unsigned here?
// C standard says it's implementation-defined
// Always use 'unsigned int' or 'signed int' explicitly
```

### 4. Endianness
```c
union { uint32_t word; uint8_t bytes[4]; } u = {.word = 0x01020304};
// bytes[0] is 0x04 (LE) or 0x01 (BE) — platform-dependent
```

---

## 8.3 Common Bugs

### Bug 1: Forgetting Struct Semicolon
```c
struct Node {
    int data;
    struct Node *next;
}     // ← Missing semicolon!
// Next declaration gets mysterious errors
int main(void) { ... }
// Error: "expected '=', ',', ';' before 'int'"
```

### Bug 2: Shallow Copy of Pointer Members
```c
typedef struct { char *data; size_t len; } Buffer;

Buffer a = {.data = malloc(100), .len = 100};
Buffer b = a;   // SHALLOW copy — both point to same malloc'd memory

free(a.data);
free(b.data);   // DOUBLE FREE! Crash or heap corruption
```

### Bug 3: Comparing Structs with memcmp (Padding)
```c
struct S { char a; int b; };
struct S x = {.a = 1, .b = 2};
struct S y = {.a = 1, .b = 2};
// Padding bytes in x and y may differ → memcmp may return non-zero
// even though all members are equal!
```

### Bug 4: Flexible Array Member Stack Allocation
```c
typedef struct { int len; int data[]; } FlexArray;
FlexArray fa;           // sizeof(fa) doesn't include data[]
fa.data[0] = 42;        // UB — no space allocated for data!
```

### Bug 5: Casting Unrelated Struct Pointers (Strict Aliasing)
```c
struct A { int x; };
struct B { int y; };

struct A a = {42};
struct B *bp = (struct B *)&a;
printf("%d\n", bp->y);  // UB! Violates strict aliasing rule
// Exception: casting to/from char* or void* is always valid
```

### Bug 6: Modifying Const Through Cast
```c
const Point cp = {1, 2};
Point *mp = (Point *)&cp;   // Casting away const
mp->x = 99;                 // UB! The object was declared const
```

---

## 8.4 Portability Concerns

### 1. Cross-Platform Struct Sizes
```c
// DON'T:
void send_struct(int fd, MyStruct *s) {
    write(fd, s, sizeof(*s));  // Padding differs between platforms!
}

// DO: Serialize field by field
void send_struct_safe(int fd, MyStruct *s) {
    uint8_t buf[7];
    buf[0] = s->type;
    memcpy(buf + 1, &s->value, 4);  // Or use htonl for endianness
    memcpy(buf + 5, &s->flags, 2);
    write(fd, buf, 7);
}
```

### 2. Packed Struct Alignment on ARM
```c
struct __attribute__((packed)) Pkt {
    uint8_t type;
    uint32_t data;  // At offset 1 — misaligned!
};

struct Pkt p;
uint32_t *dp = &p.data;  // Misaligned pointer
*dp = 42;                 // x86: works (slow). ARM: HARD FAULT!

// Fix: use memcpy
uint32_t val = 42;
memcpy(&p.data, &val, sizeof(val));  // Always safe
```

### 3. Struct Layout Across Compilers
Different compilers may pad differently. For wire protocols:
- Use `#pragma pack(1)` or `__attribute__((packed))`
- Serialize/deserialize explicitly
- Define exact byte layouts in documentation

### 4. Union Type Punning Portability
```c
// Safe in C (C99+), UB in C++
union { float f; uint32_t u; } pun;
pun.f = 3.14f;
uint32_t bits = pun.u;  // ✅ C, ❌ C++

// Safe everywhere:
float f = 3.14f;
uint32_t bits;
memcpy(&bits, &f, sizeof(bits));
```

---

## 8.5 Thread Safety with Structs

### Problem: Race Conditions
```c
typedef struct {
    int count;
    int total;
} Stats;

static Stats global_stats;  // Shared between threads

// Thread 1:
global_stats.count++;  // NOT atomic! Read-modify-write
global_stats.total += value;

// Thread 2 (same struct):
global_stats.count++;  // Race condition!
```

### Solution: Mutex / Atomic
```c
#include <stdatomic.h>

typedef struct {
    atomic_int count;
    atomic_int total;
} AtomicStats;

// Or wrap with mutex:
typedef struct {
    int count;
    int total;
    pthread_mutex_t lock;
} ProtectedStats;

void stats_increment(ProtectedStats *s, int value) {
    pthread_mutex_lock(&s->lock);
    s->count++;
    s->total += value;
    pthread_mutex_unlock(&s->lock);
}
```

### 🔧 Embedded ISR Safety
```c
// ISR writes, main loop reads — use volatile + critical section
typedef struct {
    volatile uint32_t ticks;
    volatile uint16_t adc_value;
} SharedData;

static SharedData shared;

void SysTick_Handler(void) {
    shared.ticks++;
}

uint32_t get_ticks(void) {
    uint32_t val;
    __disable_irq();   // Begin critical section
    val = shared.ticks;
    __enable_irq();    // End critical section
    return val;
}
```

---

## 8.6 The `static_assert` Safety Net (C11)

```c
#include <assert.h>

typedef struct {
    uint8_t type;
    uint16_t length;
    uint32_t payload;
} __attribute__((packed)) WireMsg;

// Compile-time checks — catch layout bugs BEFORE running
static_assert(sizeof(WireMsg) == 7, "WireMsg must be exactly 7 bytes");
static_assert(offsetof(WireMsg, length) == 1, "length must be at offset 1");
static_assert(offsetof(WireMsg, payload) == 3, "payload must be at offset 3");
// If any assertion fails → compile error with your message
```

---

## 8.7 Quick Reference: What's UB, Implementation-Defined, or Safe?

| Action | Status | Notes |
|--------|--------|-------|
| Read uninitialized struct member | **UB** | Always initialize |
| Dereference NULL struct pointer | **UB** | Check before access |
| Read inactive union member (C) | **Defined** | C99+ allows type punning |
| Read inactive union member (C++) | **UB** | Use `memcpy` instead |
| Struct assignment (=) | **Safe** | Memberwise copy |
| Struct == comparison | **Error** | Not supported by language |
| memcmp on struct with padding | **Unreliable** | Padding contents undefined |
| Packed struct pointer deref on ARM | **UB/Fault** | Use memcpy |
| Bit-field ordering | **Impl-defined** | Varies by compiler |
| `sizeof` struct | **Impl-defined** | Includes padding |
| Return local struct by value | **Safe** | Copy is made |
| Return pointer to local struct | **UB** | Dangling pointer |
| Cast between unrelated struct* | **UB** | Strict aliasing violation |
| Modify const through cast | **UB** | Never do this |

---

*Next: [Chapter 9 — Practice Problems →](09_practice.md)*
