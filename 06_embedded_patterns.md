# Chapter 6: Embedded Systems & Electronics Patterns

## 6.1 Memory-Mapped I/O (MMIO) Register Structs

In embedded systems, peripherals (GPIO, UART, SPI, Timers, DMA) are controlled through
**registers mapped to specific memory addresses**. Structs model these register blocks perfectly.

### 🔧 GPIO Register Block (STM32-style)
```c
#include <stdint.h>

typedef struct {
    volatile uint32_t MODER;    // 0x00 - Mode register
    volatile uint32_t OTYPER;   // 0x04 - Output type
    volatile uint32_t OSPEEDR;  // 0x08 - Output speed
    volatile uint32_t PUPDR;    // 0x0C - Pull-up/pull-down
    volatile uint32_t IDR;      // 0x10 - Input data (read-only)
    volatile uint32_t ODR;      // 0x14 - Output data
    volatile uint32_t BSRR;     // 0x18 - Bit set/reset (write-only)
    volatile uint32_t LCKR;     // 0x1C - Lock register
    volatile uint32_t AFRL;     // 0x20 - Alternate function low
    volatile uint32_t AFRH;     // 0x24 - Alternate function high
} GPIO_TypeDef;

// Map to hardware addresses
#define GPIOA ((GPIO_TypeDef *)0x40020000)
#define GPIOB ((GPIO_TypeDef *)0x40020400)
#define GPIOC ((GPIO_TypeDef *)0x40020800)

// Usage — Set pin 5 as output, then toggle LED
void led_init(void) {
    // Clear mode bits for pin 5 (bits [11:10]), set to 01 (output)
    GPIOA->MODER &= ~(0x3 << (5 * 2));   // Clear
    GPIOA->MODER |=  (0x1 << (5 * 2));    // Set output mode
}

void led_toggle(void) {
    GPIOA->ODR ^= (1 << 5);  // XOR toggles the bit
}

// Atomic set/reset (no read-modify-write race condition)
void led_on(void)  { GPIOA->BSRR = (1 << 5); }       // Set bit
void led_off(void) { GPIOA->BSRR = (1 << (5 + 16)); } // Reset bit
```

⚠️ **Why `volatile`?** Without it, the compiler may:
- Cache register values in CPU registers
- Optimize away "redundant" reads (but hardware changes the value!)
- Reorder register writes (breaking initialization sequences)

⚠️ **Why BSRR for atomic set/reset?**
`ODR |= (1 << 5)` is read-modify-write (3 operations). If an interrupt fires between
read and write, it could corrupt the register. `BSRR` is a single atomic write.

---

## 6.2 UART Driver Using Structs

### Complete UART Register Model and Driver
```c
typedef struct {
    volatile uint32_t SR;    // Status register
    volatile uint32_t DR;    // Data register
    volatile uint32_t BRR;   // Baud rate register
    volatile uint32_t CR1;   // Control register 1
    volatile uint32_t CR2;   // Control register 2
    volatile uint32_t CR3;   // Control register 3
    volatile uint32_t GTPR;  // Guard time and prescaler
} USART_TypeDef;

#define USART1 ((USART_TypeDef *)0x40011000)
#define USART2 ((USART_TypeDef *)0x40004400)

// Status register bit definitions
#define USART_SR_TXE  (1 << 7)   // Transmit data register empty
#define USART_SR_RXNE (1 << 5)   // Read data register not empty
#define USART_SR_TC   (1 << 6)   // Transmission complete
#define USART_SR_ORE  (1 << 3)   // Overrun error
#define USART_SR_FE   (1 << 1)   // Framing error

// Control register bits
#define USART_CR1_UE   (1 << 13)  // USART enable
#define USART_CR1_TE   (1 << 3)   // Transmitter enable
#define USART_CR1_RE   (1 << 2)   // Receiver enable
#define USART_CR1_RXNEIE (1 << 5) // RXNE interrupt enable

// Driver struct encapsulates the peripheral
typedef struct {
    USART_TypeDef *periph;
    uint32_t       baud_rate;
    uint8_t       *rx_buffer;
    uint16_t       rx_size;
    volatile uint16_t rx_head;
    volatile uint16_t rx_tail;
} UART_Driver;

void uart_init(UART_Driver *drv, USART_TypeDef *periph, uint32_t baud) {
    drv->periph = periph;
    drv->baud_rate = baud;
    drv->rx_head = 0;
    drv->rx_tail = 0;

    // Configure baud rate (assuming 16MHz clock)
    periph->BRR = 16000000 / baud;

    // Enable USART, TX, and RX
    periph->CR1 = USART_CR1_UE | USART_CR1_TE | USART_CR1_RE;
}

void uart_send_byte(UART_Driver *drv, uint8_t byte) {
    while (!(drv->periph->SR & USART_SR_TXE)) { /* wait */ }
    drv->periph->DR = byte;
}

void uart_send_string(UART_Driver *drv, const char *str) {
    while (*str) {
        uart_send_byte(drv, (uint8_t)*str++);
    }
    while (!(drv->periph->SR & USART_SR_TC)) { /* wait for completion */ }
}

int uart_receive_byte(UART_Driver *drv, uint8_t *byte, uint32_t timeout_ms) {
    uint32_t start = get_tick();  // platform-specific tick function
    while (!(drv->periph->SR & USART_SR_RXNE)) {
        if ((get_tick() - start) >= timeout_ms) return -1;  // Timeout
    }
    *byte = (uint8_t)(drv->periph->DR & 0xFF);
    return 0;
}
```

---

## 6.3 Ring Buffer (ISR-Safe) Using Structs

Critical for interrupt-driven UART, ADC, and sensor data:

```c
#include <stdint.h>
#include <stdbool.h>
#include <string.h>

#define RING_BUF_SIZE 256  // Must be power of 2!

typedef struct {
    uint8_t    buffer[RING_BUF_SIZE];
    volatile uint16_t head;  // Write position (ISR writes here)
    volatile uint16_t tail;  // Read position (main loop reads here)
} RingBuffer;

void ring_init(RingBuffer *rb) {
    rb->head = 0;
    rb->tail = 0;
    memset(rb->buffer, 0, sizeof(rb->buffer));
}

bool ring_is_empty(const RingBuffer *rb) {
    return rb->head == rb->tail;
}

bool ring_is_full(const RingBuffer *rb) {
    return ((rb->head + 1) & (RING_BUF_SIZE - 1)) == rb->tail;
}

uint16_t ring_count(const RingBuffer *rb) {
    return (rb->head - rb->tail) & (RING_BUF_SIZE - 1);
}

// Called from ISR — must be fast, no malloc, no printf
bool ring_put(RingBuffer *rb, uint8_t byte) {
    if (ring_is_full(rb)) return false;  // Overflow!
    rb->buffer[rb->head] = byte;
    rb->head = (rb->head + 1) & (RING_BUF_SIZE - 1);  // Wrap with bitmask
    return true;
}

// Called from main loop
bool ring_get(RingBuffer *rb, uint8_t *byte) {
    if (ring_is_empty(rb)) return false;
    *byte = rb->buffer[rb->tail];
    rb->tail = (rb->tail + 1) & (RING_BUF_SIZE - 1);
    return true;
}

// Usage in UART ISR:
static RingBuffer uart_rx_ring;

void USART1_IRQHandler(void) {
    if (USART1->SR & USART_SR_RXNE) {
        uint8_t byte = (uint8_t)USART1->DR;
        ring_put(&uart_rx_ring, byte);
    }
}
```

💡 **Why power-of-2 size?** The `& (SIZE - 1)` bitmask replaces expensive `% SIZE` modulo.
On ARM Cortex-M without a hardware divider, this is 1 cycle vs 10+ cycles.

---

## 6.4 State Machine with Structs

### Embedded Motor Controller State Machine
```c
typedef enum {
    STATE_IDLE,
    STATE_STARTING,
    STATE_RUNNING,
    STATE_STOPPING,
    STATE_FAULT,
    STATE_COUNT
} MotorState;

typedef enum {
    EVENT_START,
    EVENT_STOP,
    EVENT_STARTED,
    EVENT_STOPPED,
    EVENT_FAULT,
    EVENT_RESET,
    EVENT_COUNT
} MotorEvent;

typedef struct {
    MotorState current_state;
    void (*on_enter[STATE_COUNT])(void);
    void (*on_exit[STATE_COUNT])(void);
    MotorState transition_table[STATE_COUNT][EVENT_COUNT];
} StateMachine;

// State handlers
static void enter_idle(void)     { /* disable motor driver */ }
static void enter_starting(void) { /* ramp up PWM */ }
static void enter_running(void)  { /* PID control active */ }
static void enter_stopping(void) { /* ramp down PWM */ }
static void enter_fault(void)    { /* emergency stop, set fault LED */ }

static StateMachine motor_sm = {
    .current_state = STATE_IDLE,
    .on_enter = {
        [STATE_IDLE]     = enter_idle,
        [STATE_STARTING] = enter_starting,
        [STATE_RUNNING]  = enter_running,
        [STATE_STOPPING] = enter_stopping,
        [STATE_FAULT]    = enter_fault,
    },
    .transition_table = {
        //                     START          STOP           STARTED        STOPPED       FAULT         RESET
        [STATE_IDLE]     = { STATE_STARTING, STATE_IDLE,    STATE_IDLE,    STATE_IDLE,    STATE_FAULT,  STATE_IDLE },
        [STATE_STARTING] = { STATE_STARTING, STATE_STOPPING,STATE_RUNNING, STATE_IDLE,   STATE_FAULT,  STATE_IDLE },
        [STATE_RUNNING]  = { STATE_RUNNING,  STATE_STOPPING,STATE_RUNNING, STATE_RUNNING,STATE_FAULT,  STATE_RUNNING },
        [STATE_STOPPING] = { STATE_STOPPING, STATE_STOPPING,STATE_STOPPING,STATE_IDLE,   STATE_FAULT,  STATE_IDLE },
        [STATE_FAULT]    = { STATE_FAULT,    STATE_FAULT,   STATE_FAULT,   STATE_FAULT,   STATE_FAULT,  STATE_IDLE },
    },
};

void sm_process(StateMachine *sm, MotorEvent event) {
    MotorState next = sm->transition_table[sm->current_state][event];
    if (next != sm->current_state) {
        if (sm->on_exit[sm->current_state])
            sm->on_exit[sm->current_state]();
        sm->current_state = next;
        if (sm->on_enter[next])
            sm->on_enter[next]();
    }
}
```

---

## 6.5 DMA Transfer Descriptor

```c
typedef struct {
    volatile uint32_t CR;     // Control register
    volatile uint32_t NDTR;   // Number of data to transfer
    volatile uint32_t PAR;    // Peripheral address
    volatile uint32_t M0AR;   // Memory 0 address
    volatile uint32_t M1AR;   // Memory 1 address (double buffer mode)
    volatile uint32_t FCR;    // FIFO control register
} DMA_Stream_TypeDef;

// DMA config struct
typedef struct {
    DMA_Stream_TypeDef *stream;
    uint32_t periph_addr;
    uint32_t mem_addr;
    uint16_t data_count;
    enum { DMA_DIR_P2M, DMA_DIR_M2P, DMA_DIR_M2M } direction;
    enum { DMA_SIZE_BYTE, DMA_SIZE_HALF, DMA_SIZE_WORD } data_size;
    _Bool    circular;
    void     (*complete_callback)(void);
} DMA_Config;

void dma_start(const DMA_Config *cfg) {
    cfg->stream->NDTR = cfg->data_count;
    cfg->stream->PAR  = cfg->periph_addr;
    cfg->stream->M0AR = cfg->mem_addr;
    // Set direction, size, circular mode in CR...
    cfg->stream->CR |= 1;  // Enable stream
}
```

---

## 6.6 Sensor Abstraction Layer

```c
// Abstract sensor interface
typedef struct {
    const char *name;
    int  (*init)(void *config);
    int  (*read)(float *value);
    void (*deinit)(void);
    void *driver_data;  // Private driver context
} Sensor;

// Concrete implementation: Temperature (e.g., DS18B20)
static int ds18b20_init(void *cfg)   { /* configure one-wire */ return 0; }
static int ds18b20_read(float *val)  { *val = 23.5f; return 0; }
static void ds18b20_deinit(void)     { /* release bus */ }

// Concrete implementation: Accelerometer (e.g., MPU6050)
static int mpu6050_init(void *cfg)   { /* configure I2C */ return 0; }
static int mpu6050_read(float *val)  { *val = 9.81f; return 0; }
static void mpu6050_deinit(void)     { /* power down */ }

// Sensor table
static Sensor sensors[] = {
    {.name = "DS18B20",  .init = ds18b20_init, .read = ds18b20_read, .deinit = ds18b20_deinit},
    {.name = "MPU6050",  .init = mpu6050_init, .read = mpu6050_read, .deinit = mpu6050_deinit},
};

// Generic polling loop
void poll_all_sensors(void) {
    for (size_t i = 0; i < sizeof(sensors)/sizeof(sensors[0]); i++) {
        float val;
        if (sensors[i].read(&val) == 0) {
            printf("%s: %.2f\n", sensors[i].name, val);
        }
    }
}
```

---

## 6.7 RTOS Task Configuration with Structs

```c
typedef void (*TaskFunction)(void *params);

typedef struct {
    const char   *name;
    TaskFunction  function;
    uint32_t      stack_size;
    uint8_t       priority;
    void         *params;
} TaskConfig;

// Define all system tasks
static const TaskConfig system_tasks[] = {
    {"SensorTask",  sensor_task,  512,  3, NULL},
    {"CommTask",    comm_task,    1024, 2, NULL},
    {"DisplayTask", display_task, 256,  1, NULL},
    {"WatchdogTask", wdog_task,   128,  4, NULL},
};

void create_all_tasks(void) {
    for (size_t i = 0; i < sizeof(system_tasks)/sizeof(system_tasks[0]); i++) {
        const TaskConfig *t = &system_tasks[i];
        // xTaskCreate(t->function, t->name, t->stack_size, t->params, t->priority, NULL);
    }
}
```

---

## 6.8 Command Parser (CLI over UART)

```c
typedef struct {
    const char *name;
    const char *help;
    int (*handler)(int argc, char **argv);
} CLI_Command;

static int cmd_led(int argc, char **argv) {
    if (argc < 2) return -1;
    if (strcmp(argv[1], "on") == 0) led_on();
    else if (strcmp(argv[1], "off") == 0) led_off();
    else return -1;
    return 0;
}

static int cmd_reset(int argc, char **argv) {
    (void)argc; (void)argv;
    // NVIC_SystemReset();
    return 0;
}

static const CLI_Command commands[] = {
    {"led",   "led <on|off> - Control LED",  cmd_led},
    {"reset", "reset - Soft reset device",   cmd_reset},
};

void cli_execute(const char *line) {
    char buf[128];
    strncpy(buf, line, sizeof(buf) - 1);
    // Tokenize and match command...
    for (size_t i = 0; i < sizeof(commands)/sizeof(commands[0]); i++) {
        if (strcmp(buf, commands[i].name) == 0) {
            commands[i].handler(1, &(char*){buf});
            return;
        }
    }
    printf("Unknown command: %s\n", buf);
}
```

---

## Summary

| Pattern | Application |
|---------|------------|
| MMIO register structs | GPIO, UART, SPI, Timer, DMA peripheral access |
| Ring buffer | ISR-safe data buffering (UART RX, ADC samples) |
| State machine | Motor control, protocol parsing, UI navigation |
| Sensor abstraction | Pluggable driver architecture |
| DMA descriptors | Zero-copy data transfers |
| RTOS task config | Declarative task management |
| CLI command table | Debug consoles over UART |

---

*Next: [Chapter 7 — OOP Patterns in C →](07_oop_in_c.md)*
