# Chapter 7: OOP Patterns in C Using Structs

## 7.1 Encapsulation

Hide internal data behind an API. Users interact only through functions, never touching struct internals.

### header (led.h):
```c
#ifndef LED_H
#define LED_H
#include <stdint.h>
#include <stdbool.h>

// Opaque type — users only see a pointer
typedef struct LED LED;

LED  *led_create(uint8_t pin, bool active_low);
void  led_on(LED *led);
void  led_off(LED *led);
void  led_toggle(LED *led);
bool  led_is_on(const LED *led);
void  led_destroy(LED *led);

#endif
```

### implementation (led.c):
```c
#include "led.h"
#include <stdlib.h>

struct LED {
    uint8_t pin;
    bool    active_low;
    bool    state;
};

LED *led_create(uint8_t pin, bool active_low) {
    LED *led = calloc(1, sizeof(LED));
    if (led) {
        led->pin = pin;
        led->active_low = active_low;
        led->state = false;
        // gpio_set_output(pin);
    }
    return led;
}

void led_on(LED *led)  { led->state = true;  /* gpio_write(led->pin, ...) */ }
void led_off(LED *led) { led->state = false; /* gpio_write(led->pin, ...) */ }
void led_toggle(LED *led) { led->state ? led_off(led) : led_on(led); }
bool led_is_on(const LED *led) { return led->state; }
void led_destroy(LED *led) { led_off(led); free(led); }
```

Changing `struct LED` internals requires recompiling only `led.c`, not user code.

---

## 7.2 Inheritance (Struct Embedding)

C simulates inheritance by placing the "base" struct as the **first member** of the "derived" struct.
A pointer to the derived struct can be safely cast to the base type (guaranteed by C standard §6.7.2.1).

```c
// Base "class"
typedef struct {
    const char *name;
    void (*draw)(void *self);
    void (*area)(void *self, double *result);
} Shape;

// "Derived" — Circle
typedef struct {
    Shape base;      // MUST be first member
    double radius;
} Circle;

// "Derived" — Rectangle
typedef struct {
    Shape base;      // MUST be first member
    double width, height;
} Rectangle;

// Circle methods
static void circle_draw(void *self) {
    Circle *c = (Circle *)self;
    printf("Drawing circle '%s' r=%.1f\n", c->base.name, c->radius);
}
static void circle_area(void *self, double *result) {
    Circle *c = (Circle *)self;
    *result = 3.14159265 * c->radius * c->radius;
}

Circle *circle_create(const char *name, double r) {
    Circle *c = malloc(sizeof(Circle));
    if (c) {
        c->base.name = name;
        c->base.draw = circle_draw;
        c->base.area = circle_area;
        c->radius = r;
    }
    return c;
}

// Rectangle methods
static void rect_draw(void *self) {
    Rectangle *r = (Rectangle *)self;
    printf("Drawing rect '%s' %.1fx%.1f\n", r->base.name, r->width, r->height);
}
static void rect_area(void *self, double *result) {
    Rectangle *r = (Rectangle *)self;
    *result = r->width * r->height;
}

Rectangle *rect_create(const char *name, double w, double h) {
    Rectangle *r = malloc(sizeof(Rectangle));
    if (r) {
        r->base.name = name;
        r->base.draw = rect_draw;
        r->base.area = rect_area;
        r->width = w;
        r->height = h;
    }
    return r;
}
```

---

## 7.3 Polymorphism (Virtual Dispatch)

Treating different derived types through a common base pointer:

```c
int main(void) {
    // Create shapes
    Shape *shapes[3];
    shapes[0] = (Shape *)circle_create("Sun", 5.0);
    shapes[1] = (Shape *)rect_create("Window", 3.0, 4.0);
    shapes[2] = (Shape *)circle_create("Ball", 1.5);

    // Polymorphic dispatch — calls the correct function based on type
    for (int i = 0; i < 3; i++) {
        shapes[i]->draw(shapes[i]);     // Virtual call

        double a;
        shapes[i]->area(shapes[i], &a);
        printf("  Area = %.2f\n", a);
    }

    // Cleanup
    for (int i = 0; i < 3; i++) free(shapes[i]);
    return 0;
}
```

**Output:**
```
Drawing circle 'Sun' r=5.0
  Area = 78.54
Drawing rect 'Window' 3.0x4.0
  Area = 12.00
Drawing circle 'Ball' r=1.5
  Area = 7.07
```

---

## 7.4 Virtual Table (vtable) Pattern

Separate function pointers into a shared vtable to save memory when creating many objects:

```c
// Virtual table — shared by all instances of the same type
typedef struct {
    const char *type_name;
    void (*draw)(void *self);
    void (*area)(void *self, double *result);
    void (*destroy)(void *self);
} ShapeVTable;

// Base with a vtable pointer (like C++ vptr)
typedef struct {
    const ShapeVTable *vtable;  // Pointer to shared vtable
    const char *name;
} Shape;

// Circle
typedef struct {
    Shape base;
    double radius;
} Circle;

static void circle_draw(void *self)    { /* ... */ }
static void circle_area(void *self, double *r) {
    *r = 3.14159 * ((Circle*)self)->radius * ((Circle*)self)->radius;
}
static void circle_destroy(void *self) { free(self); }

// Single vtable shared by ALL circles (saves memory)
static const ShapeVTable circle_vtable = {
    .type_name = "Circle",
    .draw = circle_draw,
    .area = circle_area,
    .destroy = circle_destroy,
};

Circle *circle_new(const char *name, double r) {
    Circle *c = malloc(sizeof(Circle));
    if (c) {
        c->base.vtable = &circle_vtable;  // Point to shared vtable
        c->base.name = name;
        c->radius = r;
    }
    return c;
}

// Generic usage — the "virtual call"
void shape_draw(Shape *s)               { s->vtable->draw(s); }
void shape_area(Shape *s, double *area)  { s->vtable->area(s, area); }
void shape_destroy(Shape *s)             { s->vtable->destroy(s); }
```

**Memory comparison:**
- Without vtable: N objects × M function pointers = N×M pointers
- With vtable: N objects × 1 vtable pointer + 1 shared vtable = N+M pointers
- For 1000 circles with 3 methods: 1000×3 = 3000 vs 1000+3 = 1003 pointers

---

## 7.5 Interfaces (Abstract Base)

```c
// "Interface" — no data, just function pointers
typedef struct {
    int  (*open)(void *ctx);
    int  (*read)(void *ctx, uint8_t *buf, size_t len);
    int  (*write)(void *ctx, const uint8_t *buf, size_t len);
    void (*close)(void *ctx);
} StreamInterface;

// "Class" — holds context + implements the interface
typedef struct {
    StreamInterface iface;
    int fd;             // File descriptor (private data)
} FileStream;

typedef struct {
    StreamInterface iface;
    uint8_t *buffer;    // Memory buffer (private data)
    size_t pos;
    size_t size;
} MemoryStream;

// File implementation
static int file_open(void *ctx)  { /* open file */ return 0; }
static int file_read(void *ctx, uint8_t *buf, size_t len) {
    FileStream *fs = (FileStream *)ctx;
    return read(fs->fd, buf, len);
}
// ... write, close ...

// Memory implementation
static int mem_read(void *ctx, uint8_t *buf, size_t len) {
    MemoryStream *ms = (MemoryStream *)ctx;
    size_t avail = ms->size - ms->pos;
    size_t to_read = len < avail ? len : avail;
    memcpy(buf, ms->buffer + ms->pos, to_read);
    ms->pos += to_read;
    return (int)to_read;
}

// Generic function accepts any stream
void copy_stream(StreamInterface *src, StreamInterface *dst) {
    uint8_t buf[256];
    int n;
    while ((n = src->read(src, buf, sizeof(buf))) > 0) {
        dst->write(dst, buf, n);
    }
}
```

---

## 7.6 Constructor / Destructor Pattern

```c
typedef struct {
    int      *data;
    size_t    size;
    size_t    capacity;
} IntVector;

// Constructor
IntVector *vec_create(size_t initial_cap) {
    IntVector *v = malloc(sizeof(IntVector));
    if (!v) return NULL;
    v->data = malloc(initial_cap * sizeof(int));
    if (!v->data) { free(v); return NULL; }
    v->size = 0;
    v->capacity = initial_cap;
    return v;
}

// "Method"
int vec_push(IntVector *v, int value) {
    if (v->size >= v->capacity) {
        size_t new_cap = v->capacity * 2;
        int *new_data = realloc(v->data, new_cap * sizeof(int));
        if (!new_data) return -1;
        v->data = new_data;
        v->capacity = new_cap;
    }
    v->data[v->size++] = value;
    return 0;
}

int vec_get(const IntVector *v, size_t index) {
    if (index >= v->size) return 0;  // Or assert/abort
    return v->data[index];
}

// Destructor
void vec_destroy(IntVector *v) {
    if (v) {
        free(v->data);
        free(v);
    }
}
```

---

## Summary

| OOP Concept | C Implementation |
|-------------|-----------------|
| Encapsulation | Opaque types via forward declaration in headers |
| Inheritance | Embed base struct as first member |
| Polymorphism | Function pointers in struct (or vtable) |
| Virtual table | Shared const vtable pointer; saves memory |
| Interface | Struct of function pointers; "implemented" by concrete types |
| Constructor | `type_create()` function using `malloc` |
| Destructor | `type_destroy()` function using `free` |

---

*Next: [Chapter 8 — Edge Cases & Pitfalls →](08_edge_cases.md)*
