# Chapter 4: Union Fundamentals

## 4.1 What is a Union?

A `union` is a user-defined type where **all members share the same memory location**.
The size of a union equals the size of its **largest member**. Only ONE member
can hold a valid value at any given time.

```
STRUCT vs UNION memory layout:

struct { char a; int b; char c; }     union { char a; int b; char c; }
┌───┬───────┬───────────┬───┬───┐     ┌───────────┐
│ a │ pad   │  b (4B)   │ c │pad│     │     b     │ ← a and c share
└───┴───────┴───────────┴───┴───┘     │  a,c (1B) │   the same space
Total: 12 bytes                        └───────────┘
                                       Total: 4 bytes
```

---

## 4.2 Declaration Syntax

```c
union Data {
    int    i;      // 4 bytes
    float  f;      // 4 bytes
    char   str[8]; // 8 bytes
};
// sizeof(union Data) = 8 (largest member)

union Data d;
d.i = 42;           // 'i' is the active member
printf("%d\n", d.i); // OK — reading active member

d.f = 3.14f;         // Now 'f' is the active member; 'i' is no longer valid
printf("%f\n", d.f);  // OK
printf("%d\n", d.i);  // ⚠️ UB in strict C! (reading inactive member)
```

---

## 4.3 Initialization

```c
// Method 1: Initialize first member (C89)
union Data d1 = {42};  // Initializes 'i' (first member)

// Method 2: Designated initializer (C99+) — specify which member
union Data d2 = {.f = 3.14f};

// Method 3: Compound literal
union Data d3 = (union Data){.str = "Hello"};

// Method 4: Zero-initialize
union Data d4 = {0};  // All bytes set to 0
```

⚠️ **Edge Case — Only one member can be initialized:**
```c
union Data d = {.i = 42, .f = 3.14};  // ERROR in C11! Last one wins in some compilers
// Standard says: only one designated initializer per union
```

---

## 4.4 Union Size and Alignment

```c
union Example {
    char   c;      // 1 byte
    int    i;      // 4 bytes
    double d;      // 8 bytes
};

printf("Size: %zu\n", sizeof(union Example));   // 8 (largest member)
printf("Align: %zu\n", alignof(union Example));  // 8 (strictest alignment)
```

**All members start at the same offset (0):**
```c
#include <stddef.h>
printf("c offset: %zu\n", offsetof(union Example, c));  // 0
printf("i offset: %zu\n", offsetof(union Example, i));  // 0
printf("d offset: %zu\n", offsetof(union Example, d));  // 0
```

---

## 4.5 Practical Use Cases

### Use Case 1: Memory-Efficient Variant Data
When a value can be one of several types but only one at a time:

```c
typedef enum { TYPE_INT, TYPE_FLOAT, TYPE_STRING } DataType;

typedef struct {
    DataType type;        // Tag: tells us which union member is active
    union {
        int    int_val;
        float  float_val;
        char   str_val[32];
    } value;
} Variant;

void print_variant(const Variant *v) {
    switch (v->type) {
        case TYPE_INT:    printf("int: %d\n", v->value.int_val); break;
        case TYPE_FLOAT:  printf("float: %f\n", v->value.float_val); break;
        case TYPE_STRING: printf("string: %s\n", v->value.str_val); break;
    }
}

Variant v = {.type = TYPE_INT, .value.int_val = 42};
print_variant(&v);  // "int: 42"
```

### Use Case 2: Register Access (Embedded)
Access a hardware register as a full word or individual bytes:

```c
typedef union {
    uint32_t word;
    uint8_t  bytes[4];
    struct {
        uint16_t low;
        uint16_t high;
    } halves;
} Register32;

Register32 reg = {.word = 0xDEADBEEF};
printf("Byte 0: 0x%02X\n", reg.bytes[0]);  // 0xEF (little-endian)
printf("Low half: 0x%04X\n", reg.halves.low);  // 0xBEEF (little-endian)
```

### Use Case 3: IP Address Manipulation (Networking)
```c
typedef union {
    uint32_t addr32;       // Full 32-bit address
    uint8_t  octets[4];    // Individual octets
    struct {
        uint16_t network;  // Network portion
        uint16_t host;     // Host portion (for /16 subnet)
    } parts;
} IPv4_Address;

IPv4_Address ip = {.octets = {192, 168, 1, 100}};
printf("IP: %u.%u.%u.%u\n",
       ip.octets[0], ip.octets[1], ip.octets[2], ip.octets[3]);
printf("Raw: 0x%08X\n", ip.addr32);
```

---

## 4.6 Anonymous Unions (C11)

Union members can be accessed directly without a member name:

```c
typedef struct {
    DataType type;
    union {           // Anonymous union — no member name needed
        int    int_val;
        float  float_val;
        char   str_val[32];
    };                // No name here!
} Variant;

Variant v;
v.type = TYPE_INT;
v.int_val = 42;      // Direct access! (not v.value.int_val)
```

⚠️ **C11 required.** C89/C99 do not support anonymous unions (GCC supports them
as an extension with `-fms-extensions` or by default in GNU mode).

### Anonymous Struct inside Union
```c
typedef union {
    uint32_t raw;
    struct {               // Anonymous struct inside union
        uint8_t byte0;
        uint8_t byte1;
        uint8_t byte2;
        uint8_t byte3;
    };
} Word;

Word w = {.raw = 0xAABBCCDD};
printf("byte0 = 0x%02X\n", w.byte0);  // direct access
```

---

## 4.7 Union vs Struct — Side-by-Side

```c
struct StructVersion {
    int   a;    // offset 0,  4 bytes
    float b;    // offset 4,  4 bytes
    char  c;    // offset 8,  1 byte (+3 padding)
};
// sizeof = 12, a,b,c all exist simultaneously

union UnionVersion {
    int   a;    // offset 0,  4 bytes
    float b;    // offset 0,  4 bytes (overlaps a)
    char  c;    // offset 0,  1 byte  (overlaps a and b)
};
// sizeof = 4, only one active at a time
```

| Feature | `struct` | `union` |
|---------|----------|---------|
| Memory | Sum of members + padding | Size of largest member |
| Members active | ALL simultaneously | Only ONE at a time |
| Assignment | Copies all members | Copies all bytes (largest member size) |
| Use case | Group related data | Store alternative types |

---

## 4.8 Passing Unions to Functions

Same rules as structs — by value or by pointer:

```c
void modify_union(Register32 *reg) {
    reg->bytes[0] = 0xFF;
}

Register32 r = {.word = 0};
modify_union(&r);
printf("0x%08X\n", r.word);  // 0x000000FF (little-endian)
```

---

## 4.9 Union Arrays

```c
typedef union {
    int32_t  as_int;
    float    as_float;
    uint8_t  as_bytes[4];
} MultiView;

MultiView data[100];
data[0].as_float = 3.14f;
data[1].as_int = 42;
// Each element uses 4 bytes regardless of which member is "active"
```

---

## 4.10 Complete Example: Simple Expression Evaluator

```c
#include <stdio.h>
#include <string.h>

typedef enum { EXPR_NUM, EXPR_ADD, EXPR_MUL } ExprKind;

typedef struct Expr {
    ExprKind kind;
    union {
        double number;                      // EXPR_NUM
        struct { struct Expr *lhs, *rhs; }; // EXPR_ADD, EXPR_MUL (anonymous struct)
    };
} Expr;

Expr num(double v) {
    return (Expr){.kind = EXPR_NUM, .number = v};
}

double eval(const Expr *e) {
    switch (e->kind) {
        case EXPR_NUM: return e->number;
        case EXPR_ADD: return eval(e->lhs) + eval(e->rhs);
        case EXPR_MUL: return eval(e->lhs) * eval(e->rhs);
    }
    return 0;
}

int main(void) {
    Expr a = num(3.0);
    Expr b = num(4.0);
    Expr c = num(2.0);

    Expr add_ab = {.kind = EXPR_ADD, .lhs = &a, .rhs = &b};  // 3+4
    Expr mul_c  = {.kind = EXPR_MUL, .lhs = &add_ab, .rhs = &c}; // (3+4)*2

    printf("Result: %f\n", eval(&mul_c));  // 14.000000
    return 0;
}
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| Memory | All members share offset 0; size = largest member |
| Active member | Only ONE valid at a time; reading inactive = UB (strict C) |
| Initialization | Use designated initializers `{.member = val}` |
| Anonymous unions | C11 — direct member access without intermediate name |
| Tag + Union | "Tagged union" / "discriminated union" pattern |
| Embedded use | Register byte/word access, protocol parsing |

---

*Next: [Chapter 5 — Advanced Union Techniques →](05_union_advanced.md)*
