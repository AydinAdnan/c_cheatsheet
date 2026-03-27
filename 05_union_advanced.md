# Chapter 5: Advanced Union Techniques

## 5.1 Tagged Unions (Discriminated Unions) — The Most Important Pattern

A tagged union is a struct wrapping a **tag (discriminator)** and a **union body**.
This is the standard C way to implement sum types / variant types.

### Complete Implementation
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <stdint.h>

// The tag enum — defines all possible types
typedef enum {
    VAL_NONE = 0,
    VAL_BOOL,
    VAL_INT,
    VAL_FLOAT,
    VAL_STRING,
    VAL_ARRAY,
} ValueType;

// Forward declaration for recursive types
typedef struct Value Value;

// The tagged union
struct Value {
    ValueType type;
    union {
        _Bool    bool_val;
        int64_t  int_val;
        double   float_val;
        char    *string_val;    // heap-allocated
        struct {
            Value *items;       // dynamic array of Values
            size_t count;
        } array_val;
    };  // Anonymous union (C11)
};

// Constructor functions
Value val_none(void)      { return (Value){.type = VAL_NONE}; }
Value val_bool(_Bool b)   { return (Value){.type = VAL_BOOL, .bool_val = b}; }
Value val_int(int64_t i)  { return (Value){.type = VAL_INT, .int_val = i}; }
Value val_float(double f) { return (Value){.type = VAL_FLOAT, .float_val = f}; }

Value val_string(const char *s) {
    Value v = {.type = VAL_STRING};
    v.string_val = strdup(s);  // heap copy
    return v;
}

// Destructor — must check type to know what to free
void val_free(Value *v) {
    switch (v->type) {
        case VAL_STRING:
            free(v->string_val);
            break;
        case VAL_ARRAY:
            for (size_t i = 0; i < v->array_val.count; i++)
                val_free(&v->array_val.items[i]);
            free(v->array_val.items);
            break;
        default: break;  // No heap data
    }
    v->type = VAL_NONE;
}

// Pretty printer
void val_print(const Value *v) {
    switch (v->type) {
        case VAL_NONE:   printf("none"); break;
        case VAL_BOOL:   printf("%s", v->bool_val ? "true" : "false"); break;
        case VAL_INT:    printf("%lld", (long long)v->int_val); break;
        case VAL_FLOAT:  printf("%g", v->float_val); break;
        case VAL_STRING: printf("\"%s\"", v->string_val); break;
        case VAL_ARRAY:
            printf("[");
            for (size_t i = 0; i < v->array_val.count; i++) {
                if (i > 0) printf(", ");
                val_print(&v->array_val.items[i]);
            }
            printf("]");
            break;
    }
}
```

⚠️ **Edge Case — Forgetting to check the tag:**
```c
Value v = val_string("hello");
printf("%d\n", v.int_val);  // BUG! Reading wrong member — garbage value
// Always check v.type before accessing any union member
```

⚠️ **Edge Case — Missing cleanup:**
```c
Value v = val_string("leak me");
v = val_int(42);  // Memory leak! Old string was never freed
// Always call val_free() before reassigning
```

---

## 5.2 Type Punning via Unions

Type punning means reinterpreting the bit pattern of one type as another type.
**In C (NOT C++), reading a different union member than the one last written is DEFINED BEHAVIOR.**
(C99 §6.5.2.3, footnote 82; C11 §6.5.2.3 footnote 95.)

### Examining IEEE 754 Float Internals
```c
#include <stdio.h>
#include <stdint.h>

typedef union {
    float    f;
    uint32_t u;
    struct {
        uint32_t mantissa : 23;
        uint32_t exponent : 8;
        uint32_t sign     : 1;
    } parts;
} Float32;

void inspect_float(float value) {
    Float32 f = {.f = value};
    printf("Value:    %f\n", f.f);
    printf("Hex:      0x%08X\n", f.u);
    printf("Sign:     %u\n", f.parts.sign);
    printf("Exponent: %u (biased), %d (unbiased)\n",
           f.parts.exponent, (int)f.parts.exponent - 127);
    printf("Mantissa: 0x%06X\n", f.parts.mantissa);
    printf("\n");
}

int main(void) {
    inspect_float(1.0f);     // 0x3F800000
    inspect_float(-1.0f);    // 0xBF800000
    inspect_float(0.0f);     // 0x00000000
    inspect_float(0.1f);     // 0x3DCCCCCD
    inspect_float(1.0f/0.0f); // +Infinity: 0x7F800000
    inspect_float(0.0f/0.0f); // NaN: 0x7FC00000
    return 0;
}
```

### Fast Inverse Square Root (Quake III)
```c
float fast_inv_sqrt(float number) {
    union { float f; uint32_t i; } conv = {.f = number};
    conv.i = 0x5f3759df - (conv.i >> 1);  // Bit-level manipulation of float!
    conv.f *= 1.5f - (number * 0.5f * conv.f * conv.f);  // Newton's method
    return conv.f;
}
```

### ⚠️ Type Punning Edge Cases

**1. Union type punning is C-only; it's UB in C++:**
```c
// C:   ✅ Defined behavior (C99+)
// C++: ❌ Undefined behavior — use memcpy or std::bit_cast instead
```

**2. Endianness matters:**
```c
union {
    uint32_t word;
    uint8_t bytes[4];
} u = {.word = 0x01020304};

// Little-endian (x86, ARM): bytes = {0x04, 0x03, 0x02, 0x01}
// Big-endian (PowerPC, network byte order): bytes = {0x01, 0x02, 0x03, 0x04}
```

**3. Strict aliasing alternative (memcpy — always safe):**
```c
float f = 3.14f;
uint32_t bits;
memcpy(&bits, &f, sizeof(bits));  // Safe in both C and C++
// Compiler optimizes memcpy to a register move — zero overhead
```

---

## 5.3 Union for Endianness Handling

### 🔧 Embedded — Network Byte Order Conversion
```c
typedef union {
    uint16_t value;
    struct {
        uint8_t low;
        uint8_t high;
    } bytes;
} Uint16_LE;

// Convert little-endian to big-endian (network order)
uint16_t to_network_order(uint16_t host) {
    Uint16_LE u = {.value = host};
    return ((uint16_t)u.bytes.low << 8) | u.bytes.high;
}

// Or more portable:
uint16_t htons_portable(uint16_t h) {
    uint8_t buf[2];
    buf[0] = (h >> 8) & 0xFF;  // MSB first (big-endian / network order)
    buf[1] = h & 0xFF;
    uint16_t result;
    memcpy(&result, buf, 2);
    return result;
}
```

---

## 5.4 Unions with Bit-Fields — Hardware Register Modeling

### 🔧 Complete SPI Status Register Example
```c
typedef union {
    uint8_t raw;           // Access full register
    struct {
        uint8_t RXNE  : 1; // Bit 0: RX buffer not empty
        uint8_t TXE   : 1; // Bit 1: TX buffer empty
        uint8_t CHSIDE: 1; // Bit 2: Channel side
        uint8_t UDR   : 1; // Bit 3: Underrun flag
        uint8_t CRCERR: 1; // Bit 4: CRC error
        uint8_t MODF  : 1; // Bit 5: Mode fault
        uint8_t OVR   : 1; // Bit 6: Overrun error
        uint8_t BSY   : 1; // Bit 7: Busy flag
    } bits;
} SPI_StatusReg;

// Read register, check individual flags
volatile SPI_StatusReg *spi_sr = (SPI_StatusReg *)0x40013008;

void spi_wait_tx_empty(void) {
    while (!spi_sr->bits.TXE) { /* spin */ }
}

void spi_check_errors(void) {
    SPI_StatusReg sr = *spi_sr;  // Snapshot (single volatile read)
    if (sr.bits.OVR)    handle_overrun();
    if (sr.bits.MODF)   handle_mode_fault();
    if (sr.bits.CRCERR) handle_crc_error();
}
```

### 🔧 ARM Cortex-M Interrupt Priority Register
```c
typedef union {
    uint32_t raw;
    struct {
        uint32_t priority_group : 3;  // Bits [2:0] — preempt priority
        uint32_t sub_priority   : 5;  // Bits [7:3] — sub-priority
        uint32_t reserved       : 24; // Bits [31:8]
    } fields;
} NVIC_Priority;
```

---

## 5.5 Union of Structs — Protocol Message Parsing

### 🔧 CAN Bus Message Decoding
```c
// CAN message with 8-byte data field interpreted differently per message ID
typedef struct {
    uint32_t id;
    uint8_t  dlc;  // Data length code (0-8)
    union {
        uint8_t raw[8];

        // Engine status message (ID 0x100)
        struct {
            uint16_t rpm;
            int8_t   coolant_temp;
            uint8_t  throttle_pct;
            uint16_t fuel_pressure;
            uint8_t  reserved[2];
        } __attribute__((packed)) engine;

        // Battery message (ID 0x200)
        struct {
            uint16_t voltage_mv;
            int16_t  current_ma;
            int8_t   temperature;
            uint8_t  soc_pct;       // State of charge
            uint8_t  soh_pct;       // State of health
            uint8_t  flags;
        } __attribute__((packed)) battery;

        // GPS message (ID 0x300)
        struct {
            int32_t latitude;       // Fixed-point: degrees * 1e7
            int32_t longitude;
        } __attribute__((packed)) gps;
    } data;
} CAN_Message;

void process_can(const CAN_Message *msg) {
    switch (msg->id) {
        case 0x100:
            printf("RPM: %u, Coolant: %d°C\n",
                   msg->data.engine.rpm,
                   msg->data.engine.coolant_temp);
            break;
        case 0x200:
            printf("Battery: %u mV, %d mA, SOC: %u%%\n",
                   msg->data.battery.voltage_mv,
                   msg->data.battery.current_ma,
                   msg->data.battery.soc_pct);
            break;
        case 0x300:
            printf("GPS: %.7f, %.7f\n",
                   msg->data.gps.latitude / 1e7,
                   msg->data.gps.longitude / 1e7);
            break;
    }
}
```

---

## 5.6 Union in `struct` for Mixed-Mode Configuration

```c
typedef enum { OUTPUT_UART, OUTPUT_SPI, OUTPUT_I2C } OutputMode;

typedef struct {
    OutputMode mode;
    union {
        struct {
            uint32_t baud_rate;
            uint8_t  parity;   // 0=none, 1=odd, 2=even
            uint8_t  stop_bits;
        } uart;
        struct {
            uint32_t clock_hz;
            uint8_t  cpol;
            uint8_t  cpha;
        } spi;
        struct {
            uint8_t  address;
            uint32_t clock_hz;
            uint8_t  addr_10bit;
        } i2c;
    } config;
} OutputConfig;

OutputConfig cfg = {
    .mode = OUTPUT_SPI,
    .config.spi = {
        .clock_hz = 1000000,   // 1 MHz
        .cpol = 0,
        .cpha = 0,
    }
};
```

---

## 5.7 Unions for Safe `memcpy` / `memset` Targets

```c
typedef union {
    float    f;
    uint32_t u;
} FloatBits;

// Check if float is NaN
_Bool is_nan(float x) {
    FloatBits fb = {.f = x};
    // NaN: exponent all 1s AND mantissa non-zero
    return ((fb.u >> 23) & 0xFF) == 0xFF && (fb.u & 0x7FFFFF) != 0;
}

// Check if float is negative zero
_Bool is_neg_zero(float x) {
    FloatBits fb = {.f = x};
    return fb.u == 0x80000000;
}

// Clear sign bit (absolute value via bit manipulation)
float fast_abs(float x) {
    FloatBits fb = {.f = x};
    fb.u &= 0x7FFFFFFF;  // Clear bit 31 (sign bit)
    return fb.f;
}
```

---

## 5.8 Edge Cases and Portability Concerns

### 1. Union Assignment Copies ALL Bytes
```c
union { char c; int i; double d; } a, b;
a.d = 3.14;
b = a;  // Copies sizeof(double) bytes — entire union
```

### 2. Unions and `const`
```c
const union { int i; float f; } u = {.i = 42};
// u.i = 10;   // ERROR — const
// u.f = 3.14; // ERROR — const applies to entire union
```

### 3. Unions in Portable Code — Avoid Assuming Layout
```c
// This works on little-endian, breaks on big-endian:
union { uint32_t word; uint8_t bytes[4]; } u = {.word = 0x01020304};
// bytes[0] is 0x04 on LE, 0x01 on BE

// Portable alternative:
uint32_t word = 0x01020304;
uint8_t byte0 = (word >> 0)  & 0xFF;   // Always 0x04
uint8_t byte3 = (word >> 24) & 0xFF;   // Always 0x01
```

### 4. Unions Cannot be Compared with ==
```c
union U a = {.i = 42};
union U b = {.i = 42};
// if (a == b) {}  // COMPILE ERROR
// Must compare the active member explicitly
```

---

## Summary

| Technique | Use Case |
|-----------|----------|
| Tagged union | Variant types, JSON values, AST nodes, config |
| Type punning | IEEE 754 inspection, fast math, bit manipulation |
| Register modeling | SPI/I2C/UART status registers with bit-field access |
| Protocol parsing | CAN bus, SPI frames, network packets |
| Endianness | Network byte order, cross-platform serialization |
| Anonymous struct in union | Convenient byte/half-word access to registers |

---

*Next: [Chapter 6 — Embedded Systems Patterns →](06_embedded_patterns.md)*
