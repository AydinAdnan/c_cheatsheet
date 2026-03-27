# Chapter 2: Advanced Struct Techniques

## 2.1 Nested Structs

Structs can contain other structs as members, modeling hierarchical data naturally.

```c
typedef struct {
    float x, y, z;
} Vec3;

typedef struct {
    Vec3 position;
    Vec3 velocity;
    float mass;
} Particle;

Particle p = {
    .position = {.x = 0, .y = 1.0f, .z = 0},
    .velocity = {.x = 0.5f, .y = 0, .z = -0.1f},
    .mass = 1.5f
};

printf("pos.y = %f\n", p.position.y);  // Chained dot access
```

### Pointer to Nested Member
```c
Particle *pp = &p;
printf("vel.x = %f\n", pp->velocity.x);  // Arrow then dot
// pp->velocity.x is equivalent to (*pp).velocity.x
```

### 🔧 Embedded Example — I2C Device Configuration
```c
typedef struct {
    uint8_t address;
    uint8_t reg;
} I2C_Target;

typedef struct {
    I2C_Target target;
    uint8_t   *tx_buf;
    uint16_t   tx_len;
    uint8_t   *rx_buf;
    uint16_t   rx_len;
} I2C_Transfer;

I2C_Transfer xfer = {
    .target = {.address = 0x68, .reg = 0x3B},  // MPU6050 ACCEL_XOUT_H
    .tx_buf = &reg_addr,
    .tx_len = 1,
    .rx_buf = accel_data,
    .rx_len = 6
};
```

---

## 2.2 Arrays of Structs

```c
typedef struct {
    char name[32];
    uint8_t pin;
    uint8_t state;
} GPIO_Config;

GPIO_Config gpio_table[] = {
    {.name = "LED_RED",    .pin = 13, .state = 0},
    {.name = "LED_GREEN",  .pin = 14, .state = 0},
    {.name = "BUTTON",     .pin = 2,  .state = 1},
};

size_t num_gpios = sizeof(gpio_table) / sizeof(gpio_table[0]);

for (size_t i = 0; i < num_gpios; i++) {
    printf("Pin %u (%s): %s\n",
           gpio_table[i].pin,
           gpio_table[i].name,
           gpio_table[i].state ? "HIGH" : "LOW");
}
```

⚠️ **Edge Case — `sizeof` on pointer vs array:**
```c
void process(GPIO_Config configs[]) {
    // sizeof(configs) is sizeof(pointer), NOT the array size!
    // Arrays decay to pointers when passed to functions
    // ALWAYS pass the count separately
}
void process_safe(GPIO_Config *configs, size_t count) { ... }
```

---

## 2.3 Pointers to Structs

### Dynamic Allocation
```c
typedef struct {
    int id;
    char name[64];
    float salary;
} Employee;

// Single struct
Employee *emp = malloc(sizeof(Employee));
if (!emp) { perror("malloc"); exit(1); }
emp->id = 101;
strncpy(emp->name, "Alice", sizeof(emp->name) - 1);
emp->salary = 75000.0f;
free(emp);

// Array of structs
size_t n = 100;
Employee *team = calloc(n, sizeof(Employee));  // calloc zero-initializes
if (!team) { perror("calloc"); exit(1); }
team[0].id = 1;
// ...
free(team);
```

### Pointer Arithmetic with Structs
```c
Employee *team = calloc(5, sizeof(Employee));
Employee *third = team + 2;  // Points to team[2]
// Arithmetic uses sizeof(Employee) as the stride automatically

// Equivalent ways to access:
team[2].id;
(team + 2)->id;
(*(team + 2)).id;
```

⚠️ **Edge Case — Pointer to wrong type:**
```c
Employee *emp = malloc(sizeof(int));  // BUG! Too small
emp->id = 1;     // May work (int-sized)
emp->name[0] = 'A';  // BUFFER OVERFLOW — writing past allocation
```

---

## 2.4 Self-Referencing Structs (Linked Data Structures)

### Singly Linked List
```c
typedef struct Node {
    int data;
    struct Node *next;  // Self-reference — MUST use 'struct Node'
} Node;

// Create nodes
Node *head = malloc(sizeof(Node));
head->data = 10;
head->next = malloc(sizeof(Node));
head->next->data = 20;
head->next->next = NULL;  // End of list

// Traverse
for (Node *curr = head; curr != NULL; curr = curr->next) {
    printf("%d -> ", curr->data);
}
printf("NULL\n");  // Output: 10 -> 20 -> NULL
```

### Doubly Linked List
```c
typedef struct DNode {
    int data;
    struct DNode *prev;
    struct DNode *next;
} DNode;
```

### Binary Tree
```c
typedef struct TreeNode {
    int value;
    struct TreeNode *left;
    struct TreeNode *right;
} TreeNode;

TreeNode *new_node(int val) {
    TreeNode *n = malloc(sizeof(TreeNode));
    if (n) { *n = (TreeNode){.value = val, .left = NULL, .right = NULL}; }
    return n;
}
```

⚠️ **Edge Case — Circular reference (memory leak):**
```c
Node *a = malloc(sizeof(Node));
Node *b = malloc(sizeof(Node));
a->next = b;
b->next = a;  // Circular! Simple traversal loops forever
// free(a); free(b); — must break the cycle first
```

---

## 2.5 Function Pointers in Structs

Structs with function pointers enable **callback patterns** and rudimentary **OOP in C**.

```c
typedef struct {
    const char *name;
    int (*operate)(int a, int b);  // Function pointer member
} Operation;

int add(int a, int b) { return a + b; }
int mul(int a, int b) { return a * b; }

Operation ops[] = {
    {.name = "Add", .operate = add},
    {.name = "Multiply", .operate = mul},
};

for (int i = 0; i < 2; i++) {
    printf("%s(3,4) = %d\n", ops[i].name, ops[i].operate(3, 4));
}
// Output:
// Add(3,4) = 7
// Multiply(3,4) = 12
```

### 🔧 Embedded — Device Driver Interface (HAL Pattern)
```c
typedef struct {
    int  (*init)(void);
    int  (*read)(uint8_t *buf, size_t len);
    int  (*write)(const uint8_t *buf, size_t len);
    void (*deinit)(void);
} Driver_Interface;

// UART driver implements the interface
static int uart_init(void)   { /* configure UART registers */ return 0; }
static int uart_read(uint8_t *buf, size_t len)  { /* ... */ return len; }
static int uart_write(const uint8_t *buf, size_t len) { /* ... */ return len; }
static void uart_deinit(void) { /* disable UART */ }

const Driver_Interface uart_driver = {
    .init   = uart_init,
    .read   = uart_read,
    .write  = uart_write,
    .deinit = uart_deinit,
};

// Generic code uses the interface — doesn't know about UART specifics
void communicate(const Driver_Interface *drv) {
    drv->init();
    uint8_t buf[] = "Hello";
    drv->write(buf, sizeof(buf));
    drv->deinit();
}
```

---

## 2.6 Struct as Opaque Type (Information Hiding)

**header (sensor.h):**
```c
// Forward declaration — users see only a pointer, never the internals
typedef struct Sensor Sensor;

Sensor *sensor_create(int id, float calibration);
float   sensor_read(const Sensor *s);
void    sensor_destroy(Sensor *s);
```

**implementation (sensor.c):**
```c
#include "sensor.h"
#include <stdlib.h>

struct Sensor {      // Full definition hidden from users
    int id;
    float calibration;
    float last_reading;
};

Sensor *sensor_create(int id, float cal) {
    Sensor *s = malloc(sizeof(Sensor));
    if (s) { *s = (Sensor){.id = id, .calibration = cal}; }
    return s;
}

float sensor_read(const Sensor *s) {
    // Read from hardware, apply calibration
    float raw = /* adc_read() */ 2048.0f;
    return raw * s->calibration;
}

void sensor_destroy(Sensor *s) { free(s); }
```

**Benefit:** Changing `struct Sensor` internals does NOT require recompiling user code.
This is the C equivalent of encapsulation.

---

## 2.7 Const Structs and Const Members

```c
// Entire struct is const — no member can change
const Point origin = {.x = 0, .y = 0};
// origin.x = 5;  // COMPILE ERROR

// Pointer to const struct (read-only view)
void print(const Point *p) {
    // p->x = 5;  // COMPILE ERROR
    printf("%d\n", p->x);  // OK — reading is fine
}

// Const pointer to mutable struct
Point p = {1, 2};
Point *const fixed_ptr = &p;
fixed_ptr->x = 10;    // OK — struct is mutable
// fixed_ptr = &other; // ERROR — pointer itself is const
```

### Struct with Const Member
```c
typedef struct {
    const int id;    // Immutable after initialization
    char name[32];   // Mutable
} Record;

Record r = {.id = 42, .name = "test"};
// r.id = 99;  // COMPILE ERROR
strcpy(r.name, "updated");  // OK
```

⚠️ **Edge Case:** You CANNOT use `=` to assign to a struct with a `const` member:
```c
Record a = {.id = 1, .name = "a"};
Record b = {.id = 2, .name = "b"};
// a = b;  // COMPILE ERROR — assignment would modify const member 'id'
```

---

## 2.8 `volatile` Structs (Embedded Essential)

The `volatile` keyword tells the compiler: "this value may change at any time outside the
program's control." Essential for hardware registers and ISR-shared data.

```c
// Memory-mapped hardware register
typedef struct {
    volatile uint32_t STATUS;    // Read-only status register
    volatile uint32_t CONTROL;   // Read-write control register
    volatile uint32_t DATA;      // Data register
} UART_Registers;

// Map to hardware address
#define UART0 ((UART_Registers *)0x40013800)

void uart_send(uint8_t byte) {
    while (!(UART0->STATUS & (1 << 7))) {  // Wait for TXE flag
        // Without volatile, compiler might optimize this loop away!
    }
    UART0->DATA = byte;
}
```

⚠️ **Without `volatile`:** The compiler may:
- Cache `STATUS` in a register and never re-read from memory
- Eliminate the while loop entirely (it sees no code modifying STATUS)
- Reorder reads/writes to registers

---

## 2.9 Struct Packing into Arrays / Buffers (Serialization)

### Manual Serialization
```c
typedef struct {
    uint8_t  cmd;
    uint16_t addr;
    uint32_t value;
} Command;  // May have padding!

// Serialize to byte buffer (portable, padding-safe)
size_t serialize_cmd(const Command *cmd, uint8_t *buf) {
    size_t offset = 0;
    buf[offset++] = cmd->cmd;
    // Little-endian encoding
    buf[offset++] = (uint8_t)(cmd->addr & 0xFF);
    buf[offset++] = (uint8_t)(cmd->addr >> 8);
    buf[offset++] = (uint8_t)(cmd->value & 0xFF);
    buf[offset++] = (uint8_t)((cmd->value >> 8) & 0xFF);
    buf[offset++] = (uint8_t)((cmd->value >> 16) & 0xFF);
    buf[offset++] = (uint8_t)((cmd->value >> 24));
    return offset;  // 7 bytes, no padding
}

// Deserialize from byte buffer
Command deserialize_cmd(const uint8_t *buf) {
    Command cmd;
    cmd.cmd = buf[0];
    cmd.addr = (uint16_t)buf[1] | ((uint16_t)buf[2] << 8);
    cmd.value = (uint32_t)buf[3] | ((uint32_t)buf[4] << 8) |
                ((uint32_t)buf[5] << 16) | ((uint32_t)buf[6] << 24);
    return cmd;
}
```

---

## Summary

| Technique | Use Case |
|-----------|----------|
| Nested structs | Hierarchical data (configs, 3D objects) |
| Arrays of structs | Tables, device lists, GPIO configs |
| Self-referencing | Linked lists, trees, graphs |
| Function pointers | Callbacks, driver interfaces, state machines |
| Opaque types | API design, information hiding |
| `volatile` structs | Hardware registers, ISR-shared data |
| Serialization | Network protocols, file I/O, cross-platform |

---

*Next: [Chapter 3 — Struct Memory, Padding & Bit-Fields →](03_struct_memory.md)*
