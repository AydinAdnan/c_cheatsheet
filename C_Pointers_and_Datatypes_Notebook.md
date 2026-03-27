# C Programming: Complete Notebook on Data Types & Pointers

> A thorough, example-driven reference covering every data type and pointer concept in C — from primitive types to function pointers, memory management, and common pitfalls.

---

## Table of Contents

1. [Fundamentals: Memory & Variables](#1-fundamentals-memory--variables)
2. [Basic Data Types](#2-basic-data-types)
3. [Type Modifiers](#3-type-modifiers)
4. [Derived Data Types](#4-derived-data-types)
5. [Type Qualifiers](#5-type-qualifiers)
6. [Type Conversion & Casting](#6-type-conversion--casting)
7. [Introduction to Pointers](#7-introduction-to-pointers)
8. [Pointer Arithmetic](#8-pointer-arithmetic)
9. [Pointers and Arrays](#9-pointers-and-arrays)
10. [Pointers and Strings](#10-pointers-and-strings)
11. [Pointers and Functions](#11-pointers-and-functions)
12. [Pointers and Structs](#12-pointers-and-structs)
13. [Multi-level Pointers](#13-multi-level-pointers)
14. [Function Pointers](#14-function-pointers)
15. [Dynamic Memory Allocation](#15-dynamic-memory-allocation)
16. [Void Pointers](#16-void-pointers)
17. [Null Pointers & Dangling Pointers](#17-null-pointers--dangling-pointers)
18. [Const Pointers](#18-const-pointers)
19. [Pointers to Arrays & Array of Pointers](#19-pointers-to-arrays--array-of-pointers)
20. [Common Bugs & Best Practices](#20-common-bugs--best-practices)

---

## 1. Fundamentals: Memory & Variables

### What is Memory?

When a C program runs, the operating system provides it with memory. This memory is logically divided into segments:

```
+----------------------------+
|       Stack                |  ← local variables, function call frames
+----------------------------+
|       Heap                 |  ← dynamic allocation (malloc/calloc/realloc)
+----------------------------+
|   Uninitialized Data (BSS) |  ← global/static vars set to 0
+----------------------------+
|   Initialized Data         |  ← global/static vars with initial values
+----------------------------+
|   Text (Code)              |  ← your compiled instructions
+----------------------------+
```

### Variables and Addresses

Every variable in C occupies a contiguous block of memory. Each byte in memory has a unique **address** (a number, typically shown in hexadecimal).

```c
#include <stdio.h>

int main() {
    int x = 42;

    printf("Value   : %d\n", x);       // 42
    printf("Address : %p\n", (void*)&x); // e.g., 0x7ffeabcd1234
    printf("Size    : %zu bytes\n", sizeof(x)); // 4

    return 0;
}
```

- `&x` — the **address-of** operator, returns the memory address of `x`
- `sizeof(x)` — returns the size in bytes of the variable

---

## 2. Basic Data Types

C provides a small set of built-in (primitive) data types. Their sizes can vary by platform, but the following are typical on a 64-bit system.

### 2.1 `int` — Integer

Stores whole numbers. Typically 4 bytes.

```c
int age = 25;
int temperature = -10;
int population = 1000000;

printf("%d\n", age);         // decimal
printf("%o\n", age);         // octal
printf("%x\n", age);         // hexadecimal
printf("%05d\n", age);       // zero-padded: 00025
```

| Type       | Size    | Range (signed)                          |
|------------|---------|-----------------------------------------|
| `int`      | 4 bytes | −2,147,483,648 to 2,147,483,647         |

### 2.2 `char` — Character

Stores a single character (or a small integer). 1 byte.

```c
char letter = 'A';
char newline = '\n';
char tab = '\t';
char null_char = '\0';   // string terminator

printf("%c\n", letter);  // A
printf("%d\n", letter);  // 65 (ASCII value)

// Characters are integers — arithmetic is valid
char next = letter + 1;
printf("%c\n", next);    // B
```

**Escape sequences:**

| Escape | Meaning         |
|--------|-----------------|
| `\n`   | Newline         |
| `\t`   | Tab             |
| `\r`   | Carriage return |
| `\\`   | Backslash       |
| `\'`   | Single quote    |
| `\"`   | Double quote    |
| `\0`   | Null character  |
| `\xhh` | Hex value       |

### 2.3 `float` — Single-Precision Floating Point

4 bytes. Approximately 6–7 significant decimal digits of precision.

```c
float pi = 3.14159f;   // 'f' suffix marks float literal
float gravity = 9.81f;

printf("%.2f\n", pi);   // 3.14
printf("%e\n", pi);     // 3.141590e+00 (scientific)
printf("%g\n", pi);     // 3.14159 (shorter of %f or %e)
```

### 2.4 `double` — Double-Precision Floating Point

8 bytes. Approximately 15–16 significant digits. **Preferred over `float`** in most cases.

```c
double precise_pi = 3.14159265358979323846;
double speed_of_light = 2.998e8;  // scientific notation

printf("%.15lf\n", precise_pi);  // 3.141592653589793
```

### 2.5 `long double` — Extended Precision

10–16 bytes depending on platform and compiler. Useful for very high-precision calculations.

```c
long double ld = 3.14159265358979323846264338327950288L;
printf("%.18Lf\n", ld);
```

### 2.6 `_Bool` / `bool` — Boolean

C99 introduced `_Bool`. With `<stdbool.h>`, you can use `bool`, `true`, `false`.

```c
#include <stdbool.h>

bool is_valid = true;
bool is_done  = false;

printf("%d\n", is_valid);   // 1
printf("%d\n", is_done);    // 0

if (is_valid) {
    printf("Valid!\n");
}
```

### Complete Size/Range Reference

```c
#include <stdio.h>
#include <limits.h>   // INT_MAX, CHAR_MIN, etc.
#include <float.h>    // FLT_MAX, DBL_MAX, etc.

int main() {
    printf("char        : %zu byte  | %d to %d\n",  sizeof(char),  CHAR_MIN,  CHAR_MAX);
    printf("short       : %zu bytes | %d to %d\n",  sizeof(short), SHRT_MIN,  SHRT_MAX);
    printf("int         : %zu bytes | %d to %d\n",  sizeof(int),   INT_MIN,   INT_MAX);
    printf("long        : %zu bytes | %ld to %ld\n",sizeof(long),  LONG_MIN,  LONG_MAX);
    printf("long long   : %zu bytes | %lld to %lld\n", sizeof(long long), LLONG_MIN, LLONG_MAX);
    printf("float       : %zu bytes | %.2e to %.2e\n", sizeof(float),  FLT_MIN, FLT_MAX);
    printf("double      : %zu bytes | %.2e to %.2e\n", sizeof(double), DBL_MIN, DBL_MAX);
    return 0;
}
```

---

## 3. Type Modifiers

Modifiers alter the range or sign interpretation of basic types.

### 3.1 `signed` and `unsigned`

```c
signed int   a = -500;     // can hold negative values (default for int/char)
unsigned int b = 65000;    // only non-negative values, doubled upper range

unsigned int c = -1;       // wraps around to 4294967295 (2^32 - 1) — BEWARE
```

| Type              | Size    | Range                        |
|-------------------|---------|------------------------------|
| `signed char`     | 1 byte  | −128 to 127                  |
| `unsigned char`   | 1 byte  | 0 to 255                     |
| `signed int`      | 4 bytes | −2,147,483,648 to 2,147,483,647 |
| `unsigned int`    | 4 bytes | 0 to 4,294,967,295           |
| `unsigned long long` | 8 bytes | 0 to 18,446,744,073,709,551,615 |

### 3.2 `short` and `long`

```c
short int  s = 32000;         // 2 bytes
long  int  l = 2147483647L;   // 4 or 8 bytes (platform-dependent)
long long  ll = 9000000000LL; // guaranteed 8 bytes

// short can be written without 'int':
short x = 100;
long  y = 100000L;
```

### 3.3 Fixed-Width Integer Types (C99, `<stdint.h>`)

For portable code where exact sizes matter:

```c
#include <stdint.h>

int8_t   a = -128;           // exactly 8 bits, signed
uint8_t  b = 255;            // exactly 8 bits, unsigned
int16_t  c = -32768;
uint16_t d = 65535;
int32_t  e = -2147483648;
uint32_t f = 4294967295U;
int64_t  g = -9223372036854775807LL;
uint64_t h = 18446744073709551615ULL;

// Minimum-width types (at least N bits, but possibly wider):
int_least8_t  i = 100;
uint_least16_t j = 1000;

// Fastest types of at least N bits:
int_fast32_t k = 99999;

// Pointer-sized integer:
intptr_t  ptr_val = (intptr_t)&a;
uintptr_t uptr    = (uintptr_t)&b;
```

---

## 4. Derived Data Types

### 4.1 Arrays

A contiguous block of elements of the same type.

```c
// Declaration and initialization
int scores[5] = {95, 87, 76, 88, 91};

// Access via index (zero-based)
printf("%d\n", scores[0]);   // 95
printf("%d\n", scores[4]);   // 91

// Partial initialization — rest set to 0
int arr[5] = {1, 2};   // {1, 2, 0, 0, 0}

// Size-deduced initialization
int nums[] = {10, 20, 30};  // compiler sets size to 3

// 2D array
int matrix[3][3] = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
};
printf("%d\n", matrix[1][2]);  // 6
```

**Important:** Array name decays into a pointer to its first element in most contexts.

```c
int arr[5] = {1,2,3,4,5};
int *p = arr;          // same as &arr[0]
printf("%d\n", p[2]);  // 3
```

### 4.2 Structures (`struct`)

Groups variables of different types under one name.

```c
struct Point {
    int x;
    int y;
};

struct Student {
    char name[50];
    int  age;
    float gpa;
};

// Initialization
struct Student s1 = {"Alice", 20, 3.9f};
struct Student s2 = {.name = "Bob", .age = 22, .gpa = 3.5f};  // designated

// Member access
printf("Name: %s, Age: %d, GPA: %.1f\n", s1.name, s1.age, s1.gpa);

// typedef for cleaner syntax
typedef struct {
    double real;
    double imag;
} Complex;

Complex c = {3.0, 4.0};
printf("Complex: %.1f + %.1fi\n", c.real, c.imag);
```

**Struct size and padding:**

```c
struct Padded {
    char  a;    // 1 byte + 3 padding
    int   b;    // 4 bytes
    char  c;    // 1 byte + 3 padding
};
// sizeof = 12, not 6 (due to alignment)

struct Packed {
    int   b;    // 4 bytes
    char  a;    // 1 byte
    char  c;    // 1 byte + 2 padding
};
// sizeof = 8 (better packing — put large fields first)
```

### 4.3 Unions

All members share the same memory location. Size = size of largest member.  in C is how their members are stored in memory. A structure allocates unique memory for each member, allowing all of them to be accessed simultaneously. A union, in contrast, allocates a single, shared memory location for all its members, meaning only one member can hold a value at any given time. 

```c
union Data {
    int    i;
    float  f;
    char   str[20];
};

union Data d;
d.i = 10;
printf("int   : %d\n", d.i);  // 10

d.f = 3.14f;
printf("float : %.2f\n", d.f); // 3.14
// d.i is now meaningless (overwritten)

printf("Union size: %zu\n", sizeof(union Data));  // 20 (size of str)
```

**Use case — type punning:**

```c
union FloatBits {
    float    f;
    uint32_t bits;
};

union FloatBits fb = {.f = 1.0f};
printf("Bits of 1.0f: 0x%08X\n", fb.bits);  // 0x3F800000
```

### 4.4 Enumerations (`enum`)

Named integer constants, making code more readable.

```c
enum Day {
    SUNDAY = 0,
    MONDAY,
    TUESDAY,
    WEDNESDAY,
    THURSDAY,
    FRIDAY,
    SATURDAY
};

enum Color { RED = 1, GREEN = 2, BLUE = 4 };  // powers of 2 for bitmasks

enum Day today = WEDNESDAY;
printf("Day number: %d\n", today);  // 3

if (today == WEDNESDAY) {
    printf("Midweek!\n");
}
```

---

## 5. Type Qualifiers

### 5.1 `const`

Declares a variable whose value cannot be changed after initialization.

```c
const int MAX_SIZE = 100;
const double PI = 3.14159265358979;

// MAX_SIZE = 200;  // ERROR: assignment to const

// const with arrays
const int primes[] = {2, 3, 5, 7, 11};
// primes[0] = 1;  // ERROR
```

### 5.2 `volatile`

Tells the compiler not to optimize reads/writes — the value may change unexpectedly (e.g., hardware registers, ISR variables).

```c
volatile int sensor_value;        // could be updated by hardware
volatile int *status_reg = (volatile int *)0x4000001C;  // memory-mapped register
```

### 5.3 `restrict` (C99)

A hint to the compiler that a pointer is the only way to access the data it points to in the current scope — enables aggressive optimizations.

```c
void add_arrays(int *restrict a, int *restrict b, int *restrict c, int n) {
    for (int i = 0; i < n; i++) {
        c[i] = a[i] + b[i];  // compiler assumes a, b, c don't overlap
    }
}
```

---

## 6. Type Conversion & Casting

### 6.1 Implicit Conversion (Promotion)

C automatically converts lower-ranked types to higher-ranked ones in expressions.

```c
int    i = 10;
double d = 3.14;

double result = i + d;   // i promoted to double → 13.14
printf("%.2f\n", result);

// Integer promotion in arithmetic
char a = 200, b = 100;
int  sum = a + b;   // a and b promoted to int before addition
```

**Promotion hierarchy (low → high):**
`char` → `short` → `int` → `long` → `long long` → `float` → `double` → `long double`

### 6.2 Explicit Casting

```c
int a = 7, b = 2;
double result = (double)a / b;   // 3.5 (without cast: 3)
printf("%.1f\n", result);

// Truncation
double pi = 3.99;
int truncated = (int)pi;         // 3 (not rounded, just truncated)

// Pointer casting
int    x = 65;
char  *cp = (char *)&x;          // treat int memory as chars
printf("%c\n", *cp);             // 'A' on little-endian

// size_t to int (be careful with large values)
size_t len = 42;
int    n   = (int)len;
```

### 6.3 Integer Overflow

```c
int max = INT_MAX;         // 2147483647
int overflow = max + 1;    // undefined behavior! wraps to -2147483648

unsigned int umax = UINT_MAX;  // 4294967295
unsigned int wrap = umax + 1;  // defined: wraps to 0
```

---

## 7. Introduction to Pointers

### 7.1 What is a Pointer?

A **pointer** is a variable that stores the **memory address** of another variable, rather than a data value itself.

```
  Variable x                Pointer p
  ┌────────────┐            ┌────────────┐
  │  Value: 42 │◄───────────│ Addr: 1000 │
  │ Addr: 1000 │            │ Addr: 2000 │
  └────────────┘            └────────────┘
```

### 7.2 Declaring and Initializing Pointers

```c
int  x = 42;
int *p;         // p is a pointer to int (uninitialized — dangerous!)
p = &x;         // p now holds the address of x

// Declare and initialize together
int *q = &x;

// Multiple pointer declarations — * binds to variable, not type
int *a, *b, c;  // a and b are pointers; c is a plain int
```

### 7.3 The Two Pointer Operators

| Operator | Name        | Meaning                            |
|----------|-------------|-------------------------------------|
| `&`      | Address-of  | Returns the address of a variable   |
| `*`      | Dereference | Accesses the value at an address    |

```c
int x = 42;
int *p = &x;

printf("Address of x  : %p\n", (void*)&x);   // e.g., 0x7ffe...
printf("Value of p    : %p\n", (void*)p);     // same address
printf("Value of x    : %d\n", x);            // 42
printf("Dereferenced p: %d\n", *p);           // 42

// Modify x through p
*p = 100;
printf("x is now      : %d\n", x);            // 100
```

### 7.4 Pointer Size

Pointers always have the same size regardless of the type they point to. On a 64-bit system, all pointers are 8 bytes.

```c
printf("%zu\n", sizeof(int *));     // 8
printf("%zu\n", sizeof(char *));    // 8
printf("%zu\n", sizeof(double *));  // 8
printf("%zu\n", sizeof(void *));    // 8
```

### 7.5 Pointer to Various Types

```c
char    ch = 'Z';
float   f  = 3.14f;
double  d  = 2.718;

char   *cp = &ch;
float  *fp = &f;
double *dp = &d;

printf("char  : %c\n",  *cp);   // Z
printf("float : %.2f\n", *fp);  // 3.14
printf("double: %.3f\n", *dp);  // 2.718
```

---

## 8. Pointer Arithmetic

Since pointers hold addresses, you can perform arithmetic on them. The key rule:

> **When you add `n` to a pointer, the address advances by `n × sizeof(pointed_type)` bytes.**

```c
int arr[] = {10, 20, 30, 40, 50};
int *p = arr;   // points to arr[0]

printf("%d\n", *p);      // 10  (arr[0])
printf("%d\n", *(p+1));  // 20  (arr[1])
printf("%d\n", *(p+2));  // 30  (arr[2])

p++;                     // advance by sizeof(int) = 4 bytes
printf("%d\n", *p);      // 20

p += 2;
printf("%d\n", *p);      // 40
p--;
printf("%d\n", *p);      // 30
```

### Pointer Subtraction

```c
int arr[] = {10, 20, 30, 40, 50};
int *start = &arr[0];
int *end   = &arr[4];

ptrdiff_t diff = end - start;   // 4 (elements between, not bytes)
printf("Elements between: %td\n", diff);
```

### Comparing Pointers

```c
int arr[] = {1, 2, 3, 4, 5};
int *p = arr;
int *q = arr + 3;

if (p < q)   printf("p comes before q\n");   // true
if (p != q)  printf("different addresses\n"); // true
```

### Walking an Array with a Pointer

```c
int arr[] = {5, 10, 15, 20, 25};
int n = sizeof(arr) / sizeof(arr[0]);
int *p = arr;

for (int i = 0; i < n; i++) {
    printf("arr[%d] = %d  (addr: %p)\n", i, *(p + i), (void*)(p + i));
}
```

---

## 9. Pointers and Arrays

Arrays and pointers are closely related in C. An array name, in most contexts, **decays** to a pointer to its first element.

```c
int arr[5] = {1, 2, 3, 4, 5};

// These are equivalent:
printf("%d\n", arr[2]);     // 3
printf("%d\n", *(arr + 2)); // 3

int *p = arr;
printf("%d\n", p[2]);       // 3
printf("%d\n", *(p + 2));   // 3
```

### The Four Equivalent Notations

```c
int arr[] = {10, 20, 30};
int *p = arr;

// All four access the same element:
int val = arr[1];       // 20
val = *(arr + 1);       // 20
val = p[1];             // 20
val = *(p + 1);         // 20
```

### Key Difference: Array Name ≠ Pointer Variable

```c
int arr[5] = {1,2,3,4,5};
int *p = arr;

p++;        // OK — p is a variable
// arr++;   // ERROR — arr is not a variable, it's a fixed address

// sizeof difference:
printf("%zu\n", sizeof(arr));  // 20 (5 × 4 bytes) — full array
printf("%zu\n", sizeof(p));    // 8  (pointer size)
```

### Passing Arrays to Functions

Arrays decay to pointers when passed to functions:

```c
void print_array(int *arr, int n) {   // or: int arr[], int n
    for (int i = 0; i < n; i++) {
        printf("%d ", arr[i]);
    }
    printf("\n");
}

void double_values(int *arr, int n) {
    for (int i = 0; i < n; i++) {
        arr[i] *= 2;   // modifies original array
    }
}

int main() {
    int nums[] = {1, 2, 3, 4, 5};
    int n = sizeof(nums) / sizeof(nums[0]);

    print_array(nums, n);     // 1 2 3 4 5
    double_values(nums, n);
    print_array(nums, n);     // 2 4 6 8 10
    return 0;
}
```

### 2D Arrays and Pointers

```c
int matrix[3][4] = {
    {1, 2, 3, 4},
    {5, 6, 7, 8},
    {9,10,11,12}
};

// matrix[i][j] == *(*(matrix + i) + j)
printf("%d\n", matrix[1][2]);         // 7
printf("%d\n", *(*(matrix + 1) + 2)); // 7

// Pointer to a row (pointer to array of 4 ints)
int (*row_ptr)[4] = matrix;
printf("%d\n", row_ptr[1][2]);        // 7
```

---

## 10. Pointers and Strings

In C, strings are null-terminated character arrays. String literals are stored in read-only memory.

```c
// Array of chars — mutable
char str1[] = "Hello";
str1[0] = 'h';   // OK

// Pointer to string literal — immutable
char *str2 = "World";
// str2[0] = 'w';  // UNDEFINED BEHAVIOR (read-only memory)

// Proper mutable copy
char str3[20];
strcpy(str3, "Hello");
str3[0] = 'h';   // OK
```

### Traversing a String with a Pointer

```c
char *s = "Hello, World!";

// Method 1: index
for (int i = 0; s[i] != '\0'; i++) {
    putchar(s[i]);
}

// Method 2: pointer walk
for (char *p = s; *p != '\0'; p++) {
    putchar(*p);
}

// Method 3: while loop
char *p = s;
while (*p) {
    putchar(*p++);  // print, then advance
}
```

### String Length Without `strlen`

```c
size_t my_strlen(const char *s) {
    const char *start = s;
    while (*s++) ;          // advance until null terminator
    return s - start - 1;
}
```

### Array of Strings (Array of Char Pointers)

```c
const char *fruits[] = {"Apple", "Banana", "Cherry", "Date"};
int n = sizeof(fruits) / sizeof(fruits[0]);

for (int i = 0; i < n; i++) {
    printf("fruits[%d] = %s\n", i, fruits[i]);
}

// Accessing individual characters
printf("First char of Banana: %c\n", fruits[1][0]);  // B
printf("First char of Banana: %c\n", *(fruits[1]));  // B
```

---

## 11. Pointers and Functions

### 11.1 Pass by Value vs Pass by Pointer

```c
// Pass by value — copy made, original unchanged
void increment_val(int n) {
    n++;
}

// Pass by pointer — operate on original
void increment_ptr(int *n) {
    (*n)++;
}

int main() {
    int x = 5;
    increment_val(x);
    printf("%d\n", x);   // 5 — unchanged

    increment_ptr(&x);
    printf("%d\n", x);   // 6 — changed
    return 0;
}
```

### 11.2 Returning Multiple Values via Pointers

```c
void min_max(int *arr, int n, int *min_out, int *max_out) {
    *min_out = *max_out = arr[0];
    for (int i = 1; i < n; i++) {
        if (arr[i] < *min_out) *min_out = arr[i];
        if (arr[i] > *max_out) *max_out = arr[i];
    }
}

int main() {
    int arr[] = {3, 1, 4, 1, 5, 9, 2, 6};
    int n = sizeof(arr) / sizeof(arr[0]);
    int lo, hi;

    min_max(arr, n, &lo, &hi);
    printf("Min: %d, Max: %d\n", lo, hi);   // Min: 1, Max: 9
    return 0;
}
```

### 11.3 Returning a Pointer from a Function

```c
// WRONG — returns address of a local variable (dangling pointer)
int *bad_function() {
    int local = 42;
    return &local;   // local is destroyed after function returns
}

// CORRECT — return pointer to static variable
int *static_ptr() {
    static int value = 100;
    return &value;
}

// CORRECT — return pointer to heap memory
int *heap_ptr(int val) {
    int *p = malloc(sizeof(int));
    if (p) *p = val;
    return p;   // caller must free()
}

int main() {
    int *p = heap_ptr(42);
    printf("%d\n", *p);   // 42
    free(p);
    return 0;
}
```

---

## 12. Pointers and Structs

### 12.1 Pointer to Struct

```c
typedef struct {
    char name[50];
    int age;
    float salary;
} Employee;

Employee emp = {"Alice", 30, 75000.0f};
Employee *ep = &emp;

// Two equivalent ways to access members:
printf("Name  : %s\n", (*ep).name);   // dereference then dot
printf("Age   : %d\n", ep->age);      // arrow operator (preferred)
printf("Salary: %.2f\n", ep->salary);

// Modify through pointer
ep->age = 31;
ep->salary *= 1.1f;   // 10% raise
```

### 12.2 Array of Structs via Pointer

```c
typedef struct {
    int x, y;
} Point;

Point points[3] = {{1,2}, {3,4}, {5,6}};
Point *pp = points;

for (int i = 0; i < 3; i++) {
    printf("Point %d: (%d, %d)\n", i, pp[i].x, pp[i].y);
    // equivalent: (pp+i)->x, (pp+i)->y
}
```

### 12.3 Self-Referential Struct (Linked List Node)

```c
typedef struct Node {
    int data;
    struct Node *next;   // pointer to same type
} Node;

// Build: 1 → 2 → 3 → NULL
Node n3 = {3, NULL};
Node n2 = {2, &n3};
Node n1 = {1, &n2};

// Traverse
Node *curr = &n1;
while (curr != NULL) {
    printf("%d ", curr->data);
    curr = curr->next;
}
// Output: 1 2 3
```

---

## 13. Multi-level Pointers

### 13.1 Double Pointer (`**`)

A pointer to a pointer. Useful for modifying pointer variables from a function, and for 2D dynamic arrays.

```c
int   x  = 42;
int  *p  = &x;    // p  holds address of x
int **pp = &p;    // pp holds address of p

printf("x        = %d\n",  x);        // 42
printf("*p       = %d\n",  *p);       // 42
printf("**pp     = %d\n",  **pp);     // 42
printf("&x       = %p\n",  (void*)&x);
printf("p        = %p\n",  (void*)p); // same as &x
printf("*pp      = %p\n",  (void*)*pp); // same as p
printf("pp       = %p\n",  (void*)pp);  // address of p
```

### 13.2 Modifying a Pointer from a Function

```c
void allocate(int **pp, int value) {
    *pp = malloc(sizeof(int));  // modify the pointer itself
    if (*pp) **pp = value;
}

int main() {
    int *p = NULL;
    allocate(&p, 99);
    printf("%d\n", *p);  // 99
    free(p);
    return 0;
}
```

### 13.3 Double Pointer for 2D Dynamic Array

```c
int rows = 3, cols = 4;

// Allocate array of row pointers
int **matrix = malloc(rows * sizeof(int *));
for (int i = 0; i < rows; i++) {
    matrix[i] = malloc(cols * sizeof(int));
}

// Fill with values
for (int i = 0; i < rows; i++)
    for (int j = 0; j < cols; j++)
        matrix[i][j] = i * cols + j;

// Print
for (int i = 0; i < rows; i++) {
    for (int j = 0; j < cols; j++)
        printf("%3d", matrix[i][j]);
    printf("\n");
}

// Free
for (int i = 0; i < rows; i++) free(matrix[i]);
free(matrix);
```

### 13.4 Triple Pointer

Rarely used in practice, but valid:

```c
int x = 5;
int *p   = &x;
int **pp  = &p;
int ***ppp = &pp;

printf("%d\n", ***ppp);   // 5
```

---

## 14. Function Pointers

A function pointer stores the address of a function. This enables callbacks, dispatch tables, and polymorphism in C.

### 14.1 Basic Syntax

```c
// Declare a function pointer type: returns int, takes two ints
int (*operation)(int, int);

int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }
int mul(int a, int b) { return a * b; }

int main() {
    operation = add;
    printf("add(3,4) = %d\n", operation(3, 4));   // 7

    operation = sub;
    printf("sub(3,4) = %d\n", operation(3, 4));   // -1

    operation = mul;
    printf("mul(3,4) = %d\n", operation(3, 4));   // 12

    return 0;
}
```

### 14.2 typedef for Cleaner Syntax

```c
typedef int (*BinaryOp)(int, int);

BinaryOp ops[] = {add, sub, mul};
const char *names[] = {"add", "sub", "mul"};

for (int i = 0; i < 3; i++) {
    printf("%s(10, 3) = %d\n", names[i], ops[i](10, 3));
}
```

### 14.3 Passing Function Pointers as Arguments (Callbacks)

```c
void apply_to_array(int *arr, int n, int (*transform)(int)) {
    for (int i = 0; i < n; i++) {
        arr[i] = transform(arr[i]);
    }
}

int square(int x) { return x * x; }
int negate(int x) { return -x; }
int abs_val(int x) { return x < 0 ? -x : x; }

int main() {
    int arr[] = {-3, 1, -4, 1, 5, -9};
    int n = 6;

    apply_to_array(arr, n, abs_val);
    // arr is now {3, 1, 4, 1, 5, 9}

    apply_to_array(arr, n, square);
    // arr is now {9, 1, 16, 1, 25, 81}

    return 0;
}
```

### 14.4 `qsort` — Standard Library Callback Example

```c
#include <stdlib.h>

int cmp_asc(const void *a, const void *b) {
    return *(int*)a - *(int*)b;   // ascending
}

int cmp_desc(const void *a, const void *b) {
    return *(int*)b - *(int*)a;   // descending
}

int main() {
    int arr[] = {5, 2, 8, 1, 9, 3};
    int n = 6;

    qsort(arr, n, sizeof(int), cmp_asc);
    // {1, 2, 3, 5, 8, 9}

    qsort(arr, n, sizeof(int), cmp_desc);
    // {9, 8, 5, 3, 2, 1}

    return 0;
}
```

### 14.5 Returning a Function Pointer

```c
typedef int (*MathFunc)(int, int);

MathFunc get_operation(char op) {
    switch (op) {
        case '+': return add;
        case '-': return sub;
        case '*': return mul;
        default:  return NULL;
    }
}

int main() {
    MathFunc f = get_operation('+');
    if (f) printf("%d\n", f(10, 5));   // 15
    return 0;
}
```

---

## 15. Dynamic Memory Allocation

The C standard library provides four functions in `<stdlib.h>` for heap memory management.

### 15.1 `malloc` — Allocate Uninitialized Memory

```c
#include <stdlib.h>

// Allocate space for 5 ints
int *arr = (int *)malloc(5 * sizeof(int));

if (arr == NULL) {
    fprintf(stderr, "malloc failed\n");
    exit(EXIT_FAILURE);
}

// Initialize manually (contents are garbage otherwise)
for (int i = 0; i < 5; i++) arr[i] = i * 10;

// Use...

free(arr);   // ALWAYS free when done
arr = NULL;  // good practice: nullify after freeing
```

### 15.2 `calloc` — Allocate Zero-Initialized Memory

```c
// calloc(count, element_size) — allocates count*element_size bytes, zeroed
int *arr = (int *)calloc(5, sizeof(int));

if (!arr) { perror("calloc"); exit(1); }

// All elements are guaranteed 0
for (int i = 0; i < 5; i++) printf("%d ", arr[i]);  // 0 0 0 0 0

free(arr);
```

### 15.3 `realloc` — Resize Allocated Memory

```c
int *arr = malloc(3 * sizeof(int));
arr[0]=1; arr[1]=2; arr[2]=3;

// Grow to 5 elements
int *tmp = realloc(arr, 5 * sizeof(int));
if (!tmp) {
    free(arr);   // realloc failed — original still valid
    exit(1);
}
arr = tmp;       // NEVER do arr = realloc(arr, ...) — leaks on failure
arr[3] = 4;
arr[4] = 5;

// Shrink to 2 elements
tmp = realloc(arr, 2 * sizeof(int));
if (tmp) arr = tmp;

free(arr);
```

### 15.4 `free` — Release Memory

```c
int *p = malloc(sizeof(int));
*p = 42;
printf("%d\n", *p);

free(p);
p = NULL;   // prevents use-after-free and double-free bugs

// free(p);  // safe — freeing NULL is a no-op
```

### 15.5 Dynamic String

```c
#include <string.h>

char *make_greeting(const char *name) {
    size_t len = strlen("Hello, ") + strlen(name) + 2; // +2 for '!' and '\0'
    char *result = malloc(len);
    if (result) {
        strcpy(result, "Hello, ");
        strcat(result, name);
        strcat(result, "!");
    }
    return result;
}

int main() {
    char *msg = make_greeting("Alice");
    if (msg) {
        printf("%s\n", msg);   // Hello, Alice!
        free(msg);
    }
    return 0;
}
```

### 15.6 Memory Alignment

```c
#include <stdlib.h>  // C11

// aligned_alloc(alignment, size) — alignment must be power of 2
// size must be multiple of alignment
void *aligned = aligned_alloc(64, 128);   // 128 bytes, 64-byte aligned
// useful for SIMD operations
free(aligned);
```

### 15.7 Visualizing Heap Allocation

```
heap memory layout:
┌───────┬───────┬───────┬───────┬───────┬─────────────────────┐
│  [0]  │  [1]  │  [2]  │  [3]  │  [4]  │  (unmapped/other)   │
│  10   │  20   │  30   │  40   │  50   │                     │
└───────┴───────┴───────┴───────┴───────┴─────────────────────┘
   ↑
   arr (pointer holds this address)
```

---

## 16. Void Pointers

A `void *` is a generic pointer — it can hold the address of any type, but **cannot be dereferenced directly** (you must cast first).

```c
int    i = 10;
double d = 3.14;
char   c = 'X';

void *vp;

vp = &i;
printf("int   : %d\n",  *(int *)vp);

vp = &d;
printf("double: %.2f\n", *(double *)vp);

vp = &c;
printf("char  : %c\n",  *(char *)vp);
```

### Generic Functions with `void *`

```c
#include <string.h>

void swap(void *a, void *b, size_t size) {
    char temp[size];           // VLA or use a fixed buffer
    memcpy(temp, a,    size);
    memcpy(a,    b,    size);
    memcpy(b,    temp, size);
}

int main() {
    int x = 5, y = 10;
    swap(&x, &y, sizeof(int));
    printf("x=%d, y=%d\n", x, y);   // x=10, y=5

    double a = 1.1, b = 2.2;
    swap(&a, &b, sizeof(double));
    printf("a=%.1f, b=%.1f\n", a, b); // a=2.2, b=1.1

    return 0;
}
```

### `malloc` Returns `void *`

```c
// In C, void* is implicitly converted to any pointer type
int *p = malloc(sizeof(int));   // no cast needed in C (unlike C++)
```

---

## 17. Null Pointers & Dangling Pointers

### 17.1 Null Pointer

A pointer that points to nothing. Defined as `NULL` (typically `(void *)0`).

```c
int *p = NULL;

// Always check before dereferencing
if (p != NULL) {
    printf("%d\n", *p);
} else {
    printf("Pointer is null\n");
}

// *p;  // SEGFAULT — dereferencing NULL is undefined behavior
```

### 17.2 Dangling Pointer

A pointer that refers to memory that has been freed or gone out of scope.

```c
// Case 1: Use after free
int *p = malloc(sizeof(int));
*p = 42;
free(p);
// *p = 10;  // UNDEFINED BEHAVIOR — memory was freed
p = NULL;    // fix: nullify immediately after free

// Case 2: Pointer to local variable (out of scope)
int *bad_return() {
    int local = 99;
    return &local;   // local is on stack, destroyed after return
}

// Case 3: Iterator invalidation (realloc can move memory)
int *arr = malloc(3 * sizeof(int));
int *ref = &arr[1];        // ref points into arr
arr = realloc(arr, 100 * sizeof(int)); // arr may move — ref is now dangling!
```

### 17.3 Wild Pointer

An uninitialized pointer — contains garbage address.

```c
int *p;        // wild pointer — DO NOT use
*p = 10;       // UNDEFINED BEHAVIOR

int *p2 = NULL; // safe initialization
```

---

## 18. Const Pointers

There are two orthogonal ways to apply `const` to a pointer:

### 18.1 Pointer to `const` Data — Value is read-only

```c
int x = 10, y = 20;
const int *p = &x;   // can't change *p, but can change p itself

// *p = 99;   // ERROR — data is const
p = &y;       // OK   — pointer itself is not const
printf("%d\n", *p);  // 20
```

### 18.2 `const` Pointer — Pointer is read-only

```c
int x = 10, y = 20;
int *const p = &x;   // can't change p, but can change *p

*p = 99;      // OK   — data is not const
// p = &y;   // ERROR — pointer is const
printf("%d\n", x);   // 99
```

### 18.3 `const` Pointer to `const` Data — Both read-only

```c
int x = 10;
const int *const p = &x;   // neither p nor *p can change

// *p = 99;  // ERROR
// p = &x;   // ERROR
printf("%d\n", *p);   // 10
```

### Summary Table

| Declaration           | Can change `*p`? | Can change `p`? |
|-----------------------|------------------|-----------------|
| `int *p`              | ✅ Yes            | ✅ Yes           |
| `const int *p`        | ❌ No             | ✅ Yes           |
| `int *const p`        | ✅ Yes            | ❌ No            |
| `const int *const p`  | ❌ No             | ❌ No            |

### Using `const` in Function Parameters

```c
// Promise to caller that we won't modify the string
size_t my_strlen(const char *s) {
    size_t n = 0;
    while (*s++) n++;
    return n;
}

// Promise not to modify either array
void copy(const int *src, int *dst, int n) {
    for (int i = 0; i < n; i++) dst[i] = src[i];
}
```

---

## 19. Pointers to Arrays & Array of Pointers

### 19.1 Pointer to an Array

```c
int arr[5] = {1, 2, 3, 4, 5};

// Pointer to the whole array (note the parentheses around *p)
int (*p)[5] = &arr;     // p points to arr as a whole unit

printf("%d\n", (*p)[2]);   // 3
printf("%d\n", p[0][2]);   // 3 (same)

// p+1 advances by sizeof(int[5]) = 20 bytes
```

### 19.2 Array of Pointers

```c
int a = 1, b = 2, c = 3;
int *ptrs[3] = {&a, &b, &c};   // array of 3 int pointers

for (int i = 0; i < 3; i++) {
    printf("%d\n", *ptrs[i]);
}

// Modify through the array of pointers
*ptrs[1] = 99;
printf("b = %d\n", b);   // 99
```

### 19.3 Array of String Pointers

```c
const char *days[] = {
    "Monday", "Tuesday", "Wednesday",
    "Thursday", "Friday", "Saturday", "Sunday"
};

for (int i = 0; i < 7; i++) {
    printf("Day %d: %s\n", i+1, days[i]);
}
```

### 19.4 Sorting with Array of Pointers (avoids moving data)

```c
const char *names[] = {"Charlie", "Alice", "Eve", "Bob", "Dave"};
int n = 5;

// Bubble sort the pointer array
for (int i = 0; i < n-1; i++) {
    for (int j = 0; j < n-1-i; j++) {
        if (strcmp(names[j], names[j+1]) > 0) {
            const char *tmp = names[j];
            names[j] = names[j+1];
            names[j+1] = tmp;
        }
    }
}

for (int i = 0; i < n; i++) printf("%s\n", names[i]);
// Alice, Bob, Charlie, Dave, Eve
```

---

## 20. Common Bugs & Best Practices

### 20.1 Common Bugs

**1. Forgetting to initialize a pointer:**
```c
int *p;       // uninitialized — undefined behavior to dereference
*p = 5;       // BUG
int *p = NULL; // safe initialization
```

**2. Off-by-one in array traversal:**
```c
int arr[5];
for (int i = 0; i <= 5; i++) arr[i] = i;  // BUG: i=5 is out of bounds
for (int i = 0; i < 5;  i++) arr[i] = i;  // correct
```

**3. Memory leak — forgetting to free:**
```c
for (int i = 0; i < 1000; i++) {
    int *p = malloc(sizeof(int));
    *p = i;
    // free(p);  // FORGOT — leaks 4KB each iteration
    free(p);  // correct
}
```

**4. Double free:**
```c
int *p = malloc(sizeof(int));
free(p);
free(p);    // UNDEFINED BEHAVIOR — double free
p = NULL;   // setting to NULL after free prevents this
```

**5. Using `=` instead of `==` with pointers:**
```c
if (p = NULL)  { }  // BUG: assigns NULL to p, always false
if (p == NULL) { }  // correct
```

**6. Forgetting `NULL` terminator for strings:**
```c
char str[5] = {'H','e','l','l','o'};  // no '\0' — BUG for printf
char str[6] = {'H','e','l','l','o','\0'}; // correct
char str[]  = "Hello";                // correct — compiler adds '\0'
```

**7. Integer overflow in size calculation:**
```c
// BUG: if n is large, n*sizeof(int) could overflow size_t
int *arr = malloc(n * sizeof(int));

// Safer: check before or use calloc
if (n > SIZE_MAX / sizeof(int)) { /* handle overflow */ }
```

### 20.2 Best Practices

```c
// 1. Always initialize pointers
int *p = NULL;

// 2. Check malloc/calloc return value
int *arr = malloc(n * sizeof(int));
if (!arr) { perror("malloc"); exit(EXIT_FAILURE); }

// 3. Set pointer to NULL after free
free(arr);
arr = NULL;

// 4. Use sizeof(type) not hard-coded sizes
int *p = malloc(sizeof(*p));   // sizeof(*p) = sizeof(int)

// 5. Use const to document intent
void print_data(const int *data, int n);

// 6. Prefer ptrdiff_t for pointer differences
ptrdiff_t diff = end_ptr - start_ptr;

// 7. Use size_t for sizes and counts
size_t length = strlen(str);
for (size_t i = 0; i < length; i++) { }

// 8. Never return pointer to local variable
// BAD:  int *f() { int x=5; return &x; }
// GOOD: int *f() { static int x=5; return &x; }
//  OR:  int *f() { int *x = malloc(...); return x; }

// 9. Use valgrind (Linux) or AddressSanitizer to detect memory bugs
// Compile with: gcc -fsanitize=address -g myprogram.c

// 10. Prefer strlen+1 when copying strings
char *copy = malloc(strlen(src) + 1);  // +1 for '\0'
strcpy(copy, src);
```

### 20.3 Using AddressSanitizer

Compile your C program with:

```bash
gcc -fsanitize=address -fsanitize=undefined -g -o myprogram myprogram.c
./myprogram
```

This catches:
- Buffer overflows
- Use-after-free
- Double-free
- Memory leaks (with `-fsanitize=leak`)
- Undefined behavior (signed overflow, null dereference, etc.)

---

## Quick Reference Cheat Sheet

### Pointer Declarations

```c
int *p;           // pointer to int
int **pp;         // pointer to pointer to int
int *arr[5];      // array of 5 pointers to int
int (*arr)[5];    // pointer to array of 5 ints
int (*fp)(int);   // pointer to function taking int, returning int
const int *p;     // pointer to const int (data read-only)
int *const p;     // const pointer to int (pointer read-only)
const int *const p; // const pointer to const int (both read-only)
```

### Format Specifiers

| Type            | Specifier |
|-----------------|-----------|
| `int`           | `%d`      |
| `unsigned int`  | `%u`      |
| `long`          | `%ld`     |
| `long long`     | `%lld`    |
| `float`         | `%f`      |
| `double`        | `%lf`     |
| `long double`   | `%Lf`     |
| `char`          | `%c`      |
| `char *` (string) | `%s`    |
| `void *` (pointer) | `%p`   |
| `size_t`        | `%zu`     |
| `ptrdiff_t`     | `%td`     |
| `octal int`     | `%o`      |
| `hex int`       | `%x`/`%X` |

### Memory Functions (`<string.h>`, `<stdlib.h>`)

| Function       | Description                                 |
|----------------|---------------------------------------------|
| `malloc(n)`    | Allocate `n` bytes (uninitialized)          |
| `calloc(c,s)`  | Allocate `c*s` bytes, zero-initialized      |
| `realloc(p,n)` | Resize allocation at `p` to `n` bytes       |
| `free(p)`      | Release allocated memory                    |
| `memcpy(d,s,n)`| Copy `n` bytes from `s` to `d` (no overlap)|
| `memmove(d,s,n)`| Copy `n` bytes, handles overlap            |
| `memset(p,v,n)`| Fill `n` bytes at `p` with byte value `v`  |
| `memcmp(a,b,n)`| Compare first `n` bytes of `a` and `b`     |

---

*End of Notebook — C Programming: Data Types & Pointers*
