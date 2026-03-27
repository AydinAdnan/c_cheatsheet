# Chapter 3: Struct Memory Layout, Padding & Bit-Fields

## 3.1 Memory Alignment — Why It Matters

Modern CPUs access memory in **word-sized chunks** (4 bytes on 32-bit, 8 bytes on 64-bit).
When data is aligned to its natural boundary, the CPU reads it in a single operation.
Misaligned access may:
- Cause a **performance penalty** (x86 — 2x slower)
- Trigger a **hardware fault / bus error** (ARM Cortex-M — hard fault!)
- Be completely **unsupported** (some DSPs, older ARM cores)

### Natural Alignment Rules
| Type | Size | Alignment Requirement |
|------|------|-----------------------|
| `char` / `uint8_t` | 1 | 1 (any address) |
| `short` / `uint16_t` | 2 | 2 (even address) |
| `int` / `uint32_t` / `float` | 4 | 4 (divisible by 4) |
| `double` / `uint64_t` | 8 | 8 (divisible by 8) |
| pointer | 4 or 8 | 4 (32-bit) / 8 (64-bit) |

**Alignment of a struct** = alignment of its **largest member**.

---

## 3.2 Padding — The Invisible Bytes

The compiler inserts **padding bytes** between members (and at the end) to satisfy alignment.

### Example: Wasteful Layout
```c
struct Bad {
    char  a;    // 1 byte at offset 0
                // 3 bytes PADDING (to align b to offset 4)
    int   b;    // 4 bytes at offset 4
    char  c;    // 1 byte at offset 8
                // 3 bytes PADDING (total struct size must be multiple of 4)
};
// sizeof(struct Bad) = 12   (only 6 bytes of actual data!)
```

**Memory layout (each cell = 1 byte):**
```
Offset: 0  1  2  3  4  5  6  7  8  9  10  11
        a  .  .  .  b  b  b  b  c  .  .   .
        ^        ^              ^         ^
        data  padding          data    padding
```

### Example: Optimized Layout
```c
struct Good {
    int   b;    // 4 bytes at offset 0
    char  a;    // 1 byte at offset 4
    char  c;    // 1 byte at offset 5
                // 2 bytes PADDING (to make total = 8, multiple of 4)
};
// sizeof(struct Good) = 8   (saved 4 bytes = 33% reduction!)
```

**Memory layout:**
```
Offset: 0  1  2  3  4  5  6  7
        b  b  b  b  a  c  .  .
```

💡 **Rule: Order members from largest to smallest to minimize padding.**

---

## 3.3 `offsetof` Macro

Use `offsetof` (from `<stddef.h>`) to inspect the byte offset of each member:

```c
#include <stddef.h>

struct Example {
    char  a;
    int   b;
    short c;
};

printf("a at offset %zu\n", offsetof(struct Example, a));  // 0
printf("b at offset %zu\n", offsetof(struct Example, b));  // 4 (after 3 padding bytes)
printf("c at offset %zu\n", offsetof(struct Example, c));  // 8
printf("Total size: %zu\n", sizeof(struct Example));        // 12
```

### 🔧 Embedded Use — Container-Of Pattern (Linux Kernel)
```c
#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))

struct device {
    int id;
    struct list_node node;  // Embedded in a linked list
    char name[32];
};

// Given a pointer to the 'node' member, get back the 'device':
struct device *dev = container_of(node_ptr, struct device, node);
```
This is one of the most important patterns in the Linux kernel.

---

## 3.4 Struct Packing (`__attribute__((packed))` / `#pragma pack`)

Packing removes ALL padding, forcing members to be tightly packed.

### GCC/Clang
```c
struct __attribute__((packed)) PackedMsg {
    uint8_t  type;      // offset 0
    uint32_t timestamp; // offset 1 (misaligned!)
    uint16_t length;    // offset 5
    uint8_t  checksum;  // offset 7
};
// sizeof = 8 (no padding at all)
```

### MSVC / Cross-compiler
```c
#pragma pack(push, 1)
struct PackedMsg {
    uint8_t  type;
    uint32_t timestamp;
    uint16_t length;
    uint8_t  checksum;
};
#pragma pack(pop)  // Restore default packing
```

### ⚠️ Dangers of Packing

**1. Performance hit on x86:**
Misaligned access works but is ~2x slower for misaligned 4-byte reads.

**2. Hard fault on ARM:**
```c
struct __attribute__((packed)) Bad {
    uint8_t a;
    uint32_t b;  // At offset 1 — misaligned!
};

struct Bad s = {.a = 1, .b = 0x12345678};
uint32_t *p = &s.b;   // Pointer to misaligned address
uint32_t val = *p;     // ARM HARD FAULT! (on many Cortex-M cores)
```

**3. Taking address of packed member:**
```c
struct __attribute__((packed)) Pkt {
    uint8_t type;
    uint32_t data;
};

struct Pkt pkt;
uint32_t *dp = &pkt.data;  // WARNING: pointer to misaligned member
// Using *dp may crash on ARM
```

💡 **Safe packed access pattern:**
```c
uint32_t val;
memcpy(&val, &pkt.data, sizeof(val));  // memcpy handles alignment safely
```

### When to Pack
- Network protocol headers that must match wire format exactly
- Binary file format structures
- Hardware register maps (but prefer `volatile` + correct alignment instead)

### When NOT to Pack
- General application structs
- Performance-critical hot paths
- Structs used only internally

---

## 3.5 Bit-Fields

Bit-fields let you specify the exact number of bits for each member, enabling sub-byte packing.

### Basic Syntax
```c
struct Flags {
    unsigned int ready   : 1;   // 1 bit
    unsigned int error   : 1;   // 1 bit
    unsigned int mode    : 3;   // 3 bits (values 0-7)
    unsigned int channel : 4;   // 4 bits (values 0-15)
    unsigned int padding : 23;  // Fill remaining bits
};
// sizeof = 4 (fits in one uint32_t)
```

### Access Like Normal Members
```c
struct Flags f = {0};
f.ready = 1;
f.mode = 5;
f.channel = 12;

if (f.error) { /* handle error */ }
```

### 🔧 Embedded — Hardware Register Mapping
```c
// STM32 GPIO Mode Register (simplified)
typedef struct {
    uint32_t MODE0  : 2;   // Pin 0 mode (00=input, 01=output, 10=alt, 11=analog)
    uint32_t MODE1  : 2;   // Pin 1 mode
    uint32_t MODE2  : 2;   // Pin 2 mode
    uint32_t MODE3  : 2;
    uint32_t MODE4  : 2;
    uint32_t MODE5  : 2;
    uint32_t MODE6  : 2;
    uint32_t MODE7  : 2;
    uint32_t MODE8  : 2;
    uint32_t MODE9  : 2;
    uint32_t MODE10 : 2;
    uint32_t MODE11 : 2;
    uint32_t MODE12 : 2;
    uint32_t MODE13 : 2;
    uint32_t MODE14 : 2;
    uint32_t MODE15 : 2;
} GPIO_MODER_Bits;

// Usage:
volatile GPIO_MODER_Bits *moder = (GPIO_MODER_Bits *)0x48000000;
moder->MODE5 = 0x01;  // Set pin 5 as general purpose output
```

### ⚠️ Bit-Field Edge Cases & Pitfalls

**1. Bit ordering is implementation-defined:**
The C standard does NOT specify whether bits are assigned from LSB or MSB.
GCC on x86/ARM: LSB first. Other compilers may differ.
```c
struct {
    uint8_t low  : 4;
    uint8_t high : 4;
};
// On GCC: low = bits 0-3, high = bits 4-7
// On another compiler: could be reversed!
```

**2. You CANNOT take the address of a bit-field:**
```c
struct Flags f;
&f.ready;  // COMPILE ERROR — bit-fields don't have byte addresses
```

**3. Bit-field width cannot exceed the type:**
```c
struct {
    uint8_t val : 9;  // ERROR — uint8_t is only 8 bits
};
```

**4. Signed bit-fields are tricky:**
```c
struct {
    int val : 3;  // Range: -4 to +3 (NOT 0 to 7!)
};
// Use 'unsigned int' for bit-fields unless you need signed values
```

**5. Zero-width bit-field forces next field to next unit boundary:**
```c
struct {
    uint32_t a : 5;
    uint32_t   : 0;   // Force alignment to next uint32_t boundary
    uint32_t b : 3;    // Starts in a new 32-bit word
};
```

**6. Unnamed bit-fields (reserved bits):**
```c
struct StatusReg {
    uint32_t ready    : 1;
    uint32_t          : 3;   // 3 reserved/unused bits
    uint32_t error    : 1;
    uint32_t          : 27;  // Remaining bits unused
};
```

---

## 3.6 Alignment Control (`_Alignas` / `alignas` — C11+)

```c
#include <stdalign.h>

struct alignas(16) AlignedData {
    float values[4];
};
// Address of any AlignedData variable will be a multiple of 16

// Also works on members:
struct CacheLine {
    alignas(64) char data[64];  // Aligned to cache line boundary
};
```

### 🔧 Embedded — DMA Buffer Alignment
DMA controllers often require buffers to be aligned to specific boundaries:
```c
// DMA requires 4-byte aligned buffers on most ARM MCUs
static uint8_t dma_buffer[256] __attribute__((aligned(4)));

// Or using C11:
static alignas(4) uint8_t dma_buffer[256];
```

---

## 3.7 Checking Alignment with `_Alignof` / `alignof`

```c
#include <stdalign.h>

printf("int alignment: %zu\n", alignof(int));        // 4
printf("double alignment: %zu\n", alignof(double));   // 8

struct Mixed { char c; double d; };
printf("Mixed alignment: %zu\n", alignof(struct Mixed));  // 8 (largest member)
```

---

## 3.8 Practical: Analyzing Memory Layout

```c
#include <stdio.h>
#include <stddef.h>
#include <stdint.h>

#define SHOW_LAYOUT(type, member) \
    printf("  %-15s  offset: %2zu  size: %zu\n", \
           #member, offsetof(type, member), sizeof(((type*)0)->member))

typedef struct {
    uint8_t   a;
    uint32_t  b;
    uint8_t   c;
    uint16_t  d;
    uint64_t  e;
} LayoutDemo;

int main(void) {
    printf("struct LayoutDemo (size=%zu, align=%zu):\n",
           sizeof(LayoutDemo), alignof(LayoutDemo));
    SHOW_LAYOUT(LayoutDemo, a);
    SHOW_LAYOUT(LayoutDemo, b);
    SHOW_LAYOUT(LayoutDemo, c);
    SHOW_LAYOUT(LayoutDemo, d);
    SHOW_LAYOUT(LayoutDemo, e);
    return 0;
}
```

**Likely output (64-bit system):**
```
struct LayoutDemo (size=24, align=8):
  a                offset:  0  size: 1
  b                offset:  4  size: 4
  c                offset:  8  size: 1
  d                offset: 10  size: 2
  e                offset: 16  size: 8
```

**Optimized reordering: `(e, b, d, c, a)` → size=16 instead of 24.**

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| Alignment | Types must sit at addresses divisible by their size |
| Padding | Compiler inserts invisible bytes; order large→small to minimize |
| `offsetof` | Inspect member offsets; enables `container_of` pattern |
| Packing | Removes padding; use for wire formats; dangerous on ARM |
| Bit-fields | Sub-byte fields; bit order is implementation-defined |
| `alignas` | Force specific alignment (DMA, cache lines, SIMD) |

---

*Next: [Chapter 4 — Union Fundamentals →](04_union_basics.md)*
