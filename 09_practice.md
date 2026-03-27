# Chapter 9: Practice Problems

## 🟢 Beginner

### Problem 1: Student Record
Create a `Student` struct with `name` (char[50]), `roll_number` (int), `marks[5]` (float),
and `average` (float). Write functions to:
- Input student data
- Calculate and store the average
- Print the student record
- Find the student with the highest average in an array of students

<details><summary>💡 Hint</summary>
Use `fgets` for name input. Calculate average = sum / 5. Pass struct by pointer.
</details>

---

### Problem 2: Complex Numbers
Implement a `Complex` struct with `real` and `imag` (double). Write:
- `complex_add(a, b)` → returns sum
- `complex_mul(a, b)` → returns product (`(a+bi)(c+di) = (ac-bd) + (ad+bc)i`)
- `complex_magnitude(c)` → returns `sqrt(real² + imag²)`
- `complex_print(c)` → prints as `"3.0 + 4.0i"` or `"3.0 - 2.0i"`

---

### Problem 3: Date Calculator
Struct `Date` with `day`, `month`, `year`. Write:
- `date_is_valid(d)` — validate including leap years
- `date_diff_days(d1, d2)` — days between two dates
- `date_add_days(d, n)` — add n days to a date
- `date_day_of_week(d)` — return "Monday", "Tuesday", etc.

---

## 🟡 Intermediate

### Problem 4: Linked List with Struct
Implement a singly linked list with full operations:
```c
typedef struct Node { int data; struct Node *next; } Node;

Node *list_create(void);
void  list_push_front(Node **head, int data);
void  list_push_back(Node **head, int data);
int   list_pop_front(Node **head);
Node *list_find(Node *head, int data);
void  list_delete(Node **head, int data);
void  list_reverse(Node **head);
void  list_print(const Node *head);
void  list_destroy(Node **head);
```
⚠️ Handle: empty list, single node, deleting head, memory leaks.

---

### Problem 5: Padding Analysis Tool
Write a program that accepts struct declarations and outputs:
- Each member's offset, size, and alignment
- Total padding bytes and percentage wasted
- Suggested optimal member ordering

Test with at least 5 different struct layouts.

---

### Problem 6: Mini JSON Parser
Define a tagged union for JSON values:
```c
typedef enum { JSON_NULL, JSON_BOOL, JSON_NUMBER, JSON_STRING, JSON_ARRAY, JSON_OBJECT } JsonType;

typedef struct JsonValue {
    JsonType type;
    union {
        _Bool bool_val;
        double number_val;
        char *string_val;
        struct { struct JsonValue *items; size_t count; } array_val;
        struct { char **keys; struct JsonValue *values; size_t count; } object_val;
    };
} JsonValue;
```
Implement: `json_print(v)`, `json_free(v)`, and constructors for each type.
Bonus: parse a JSON string into this structure.

---

### Problem 7: Expression Tree Evaluator
Using union-based AST nodes from Chapter 5's example, extend to support:
- Subtraction, division
- Unary negation
- Parenthesized expressions
- Variables (lookup from a symbol table)
- Parse from string like `"(3 + x) * 2"`

---

## 🔴 Advanced

### Problem 8: Generic Dynamic Array (void * based)
```c
typedef struct {
    void   *data;         // Generic data buffer
    size_t  elem_size;    // Size of each element
    size_t  length;       // Number of elements
    size_t  capacity;     // Allocated capacity
    int   (*compare)(const void *, const void *);  // Comparison function
} GenericArray;

GenericArray *ga_create(size_t elem_size, int (*cmp)(const void*, const void*));
void  ga_push(GenericArray *ga, const void *elem);
void *ga_get(const GenericArray *ga, size_t index);
void  ga_sort(GenericArray *ga);
int   ga_search(const GenericArray *ga, const void *key);  // Binary search
void  ga_destroy(GenericArray *ga);
```
Test with `int`, `double`, and custom structs.

---

### Problem 9: Memory Pool Allocator
Implement a fixed-size block allocator using a struct:
```c
typedef struct {
    uint8_t *pool;           // Pre-allocated memory
    size_t   block_size;     // Size of each block
    size_t   total_blocks;   // Total number of blocks
    size_t   used_blocks;    // Currently allocated
    uint32_t *bitmap;        // Allocation bitmap (1=used, 0=free)
} MemPool;

MemPool *mempool_create(size_t block_size, size_t count);
void    *mempool_alloc(MemPool *mp);
void     mempool_free(MemPool *mp, void *ptr);
void     mempool_stats(const MemPool *mp);
void     mempool_destroy(MemPool *mp);
```
Requirements: O(1) alloc and free. Validate pointer belongs to pool on free.

---

### Problem 10: 🔧 SPI Flash Driver (Embedded)
Design a complete SPI Flash driver using struct patterns from Chapter 6:
```c
typedef struct {
    // Hardware interface
    void (*cs_assert)(void);
    void (*cs_deassert)(void);
    uint8_t (*spi_transfer)(uint8_t byte);

    // Device info
    uint32_t capacity;     // Total bytes
    uint16_t page_size;    // Write page size
    uint16_t sector_size;  // Erase sector size

    // State
    _Bool initialized;
} SPIFlash;

int  spiflash_init(SPIFlash *dev);
int  spiflash_read(SPIFlash *dev, uint32_t addr, uint8_t *buf, size_t len);
int  spiflash_write_page(SPIFlash *dev, uint32_t addr, const uint8_t *data, size_t len);
int  spiflash_erase_sector(SPIFlash *dev, uint32_t addr);
int  spiflash_read_id(SPIFlash *dev, uint8_t *mfr, uint16_t *device);
void spiflash_wait_busy(SPIFlash *dev);
```

---

### Problem 11: 🔧 Protocol Codec (Embedded)
Implement a MODBUS RTU frame encoder/decoder:
```c
typedef struct __attribute__((packed)) {
    uint8_t  slave_addr;
    uint8_t  function_code;
    uint16_t start_register;
    uint16_t register_count;
    uint16_t crc16;
} ModbusRequest;

typedef struct {
    uint8_t  slave_addr;
    uint8_t  function_code;
    uint8_t  byte_count;
    uint8_t  data[252];  // Max MODBUS data
    uint16_t crc16;
} ModbusResponse;
```
Handle: CRC calculation, byte-order conversion (big-endian on wire),
error responses, timeout detection.

---

### Problem 12: 🔧 Finite State Machine Framework
Build a reusable FSM framework:
```c
typedef struct StateMachine StateMachine;
typedef void (*StateHandler)(StateMachine *sm, int event);

typedef struct {
    const char   *name;
    StateHandler  on_enter;
    StateHandler  on_event;
    StateHandler  on_exit;
} State;

struct StateMachine {
    const State  *states;
    size_t        state_count;
    size_t        current;
    void         *user_data;
    void        (*log)(const char *msg);  // Optional logging
};
```
Use it to implement: traffic light controller, vending machine, or UART protocol parser
with states: IDLE → HEADER → LENGTH → DATA → CRC → COMPLETE.

---

## Challenge Problems

### Challenge 1: Implement a Hash Map
```c
typedef struct Entry { char *key; void *value; struct Entry *next; } Entry;
typedef struct {
    Entry   **buckets;
    size_t    capacity;
    size_t    count;
    size_t  (*hash)(const char *key);
} HashMap;
```
Handle: collision resolution (chaining), dynamic resizing, iteration.

### Challenge 2: Build a Mini Database
Define table schemas using structs of structs. Implement:
- Define schema (column names, types)
- INSERT, SELECT, UPDATE, DELETE operations
- WHERE clauses using function pointers
- Store data in a binary file using serialized structs

### Challenge 3: 🔧 CAN Bus Diagnostic Tool
Parse and display CAN bus messages using the patterns from Chapter 5:
- Define message database as array of descriptor structs
- Decode signals from raw CAN data (bit extraction with position and length)
- Handle both little-endian and big-endian signals
- Display in real-time with formatting

---

## Tips for Practice

1. **Compile with warnings:** `gcc -Wall -Wextra -Wpedantic -std=c11`
2. **Use sanitizers:** `gcc -fsanitize=address,undefined` catches memory bugs
3. **Use Valgrind:** `valgrind ./program` detects memory leaks
4. **Draw memory layouts** on paper before coding struct-heavy programs
5. **Test edge cases:** empty inputs, NULL pointers, overflow, boundary values
6. **Profile:** `sizeof` everything, verify with `offsetof` and `static_assert`

---

*← Back to [Course Index](00_course_index.md)*
