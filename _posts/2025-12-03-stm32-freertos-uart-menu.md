---
title: 'RTOS Systems (Part 2): FreeRTOS LED Control with UART Menu'
date: 2025-12-03 12:00:00 -0800
permalink: /posts/2025/12/03/stm32-freertos-uart-menu
tags:
  - embedded systems
  - stm32
  - freertos
  - rtos
  - uart
redirect_from:
  - /2025/12/03/stm32-freertos-uart-menu.html
toc: true
---

---

**← Previous:** [Part 1: Multi-Task LED Blinker Without RTOS](/posts/2025/09/02/stm32-led-blinker-bare-metal)

**Next in Series →** [Part 3: ESP8266 Wi-Fi Web Server](/posts/2025/12/07/esp8266-wifi-web-server)

---

*This series covers RTOS systems development, progressing from bare-metal programming to integrated IoT systems.*

- [Part 1: Multi-Task LED Blinker Without RTOS][part1]
- [Part 2: FreeRTOS LED Control with UART Menu][part2] (this post)
- [Part 3: ESP8266 Wi-Fi Web Server][part3]
- [Part 4: STM32 + ESP8266 Integrated IoT LED Controller][part4]

## Project Overview

After experiencing the limitations of bare-metal programming in Part 1, I rebuilt the LED controller using **FreeRTOS** - a real-time operating system that brings proper task management, scheduling, and inter-task communication to embedded systems.

**GitHub Repository:** [rtos-led-control-uart-menu](https://github.com/sharan-naribole/rtos-led-control-uart-menu)

**Key Innovation:** A **dedicated print task** with a message queue for thread-safe, non-blocking debug logging.

## Why FreeRTOS?

In Part 1's bare-metal implementation, I hit these limitations:

❌ **Busy waiting** - `delay_ms()` wasted CPU cycles
❌ **No task priorities** - All code ran in main loop sequentially
❌ **Manual coordination** - Had to track timing for 4 LEDs manually
❌ **Poor scalability** - Adding a 5th task would complicate the code significantly

**FreeRTOS solves these:**

✅ **Pre-emptive scheduling** - Higher priority tasks run immediately
✅ **Blocking delays** - `vTaskDelay()` yields CPU to other tasks
✅ **Software timers** - LED patterns managed by RTOS timer service
✅ **Thread-safe communication** - Queues, semaphores, mutexes built-in
✅ **Power efficiency** - IDLE task enters WFI (Wait For Interrupt) sleep mode

## Key Features

**Interactive UART Menu System:**
```
========================================
      LED Pattern Control Menu
========================================
Main Menu:
  0. Display current LED pattern
  1. LED Pattern Menu
  2. Exit

Select option (0-2):
```

**4 LED Patterns (Software Timers):**
- Pattern 1: All LEDs solid ON
- Pattern 2: Green (100ms blink), Orange (1000ms blink) - Different frequencies
- Pattern 3: Green + Orange synchronized (100ms blink)
- Pattern 4: All LEDs OFF

**Advanced Features:**
- 🔒 Thread-safe UART TX via dedicated print task (no mutexes!)
- 📨 Efficient UART RX with stream buffer (zero CPU polling)
- 🐕 Watchdog system detecting hung/deadlocked tasks
- ⚡ Non-blocking print operations (~20-50μs latency)
- 💤 Power-efficient idle hook with WFI instruction

## Architecture Overview

### Task Hierarchy

```
Priority 4: Watchdog Task (256 words stack)
            └─ Monitors UART_Task, CMD_Handler, Print_Task
            └─ Alerts if any task hasn't "fed" watchdog in 5 seconds

Priority 3: Print_Task (512 words stack)
            └─ Exclusive UART TX owner
            └─ Message queue: 10 entries × 512 bytes
            └─ Blocks until messages available

Priority 2: UART_Task (256 words stack)
            └─ Character reception via stream buffer
            └─ Wakes instantly on RX interrupt
            └─ Feeds characters to command handler

Priority 2: CMD_Handler (256 words stack)
            └─ Menu state machine
            └─ LED pattern selection
            └─ Non-blocking command processing

Priority 2: Timer Service Task (configTIMER_TASK_STACK_DEPTH)
            └─ Software timer callbacks for LED patterns
            └─ Controls 4 LEDs on PD12-PD15

Priority 0: Idle Task
            └─ Power save (executes WFI instruction)
```

### Print Task Innovation

**Traditional Mutex Approach (What I Didn't Do):**

```c
// Problem: All tasks compete for UART
void task_A(void *params) {
    xSemaphoreTake(uart_mutex, portMAX_DELAY);  // Might block!
    HAL_UART_Transmit(&huart2, "Task A\r\n", 8, 100);  // Task blocked here
    xSemaphoreGive(uart_mutex);
}

void task_B(void *params) {
    xSemaphoreTake(uart_mutex, portMAX_DELAY);  // Might block!
    HAL_UART_Transmit(&huart2, "Task B\r\n", 8, 100);  // Task blocked here
    xSemaphoreGive(uart_mutex);
}
```

**Issues:**
- Priority inversion (low-priority task holds mutex, blocks high-priority task)
- Blocking delays (task stuck waiting for UART transmission to complete)
- Mutex boilerplate code repeated in 10+ locations

**My Solution - Dedicated Print Task:**

```c
// Application code (clean and simple!)
void task_A(void *params) {
    print_message("Task A running\r\n");  // Returns immediately!
    // Continue execution without waiting
}

void task_B(void *params) {
    print_message("Task B running\r\n");  // Returns immediately!
    // Continue execution without waiting
}

// Behind the scenes (print_task.c)
BaseType_t print_message(const char *message) {
    // Enqueue message (20-50μs) and return
    return xQueueSend(print_queue, message, pdMS_TO_TICKS(100));
}

void print_task_handler(void *parameters) {
    char buffer[512];
    while (1) {
        // Block until message available (yields CPU)
        if (xQueueReceive(print_queue, buffer, portMAX_DELAY) == pdTRUE) {
            // Exclusive UART access (no mutex needed!)
            HAL_UART_Transmit(&huart2, buffer, strlen(buffer), HAL_MAX_DELAY);
        }
    }
}
```

**Benefits:**
- ✅ **Non-blocking** - `print_message()` returns in ~30μs
- ✅ **No priority inversion** - Queue-based synchronization is priority-aware
- ✅ **FIFO ordering** - Messages printed in order received
- ✅ **Single point of control** - Easy to add features (timestamps, log levels, etc.)
- ✅ **Clean application code** - No mutex boilerplate scattered everywhere

**Performance Measured:**
```
print_message() latency: 25μs average (queue send operation)
vs. Mutex approach: 100-200μs blocking time per print
```

This architecture proved robust - I reused it in Part 4 for the integrated IoT system!

## UART RX with Stream Buffer

**Efficient interrupt-driven reception:**

```c
// ISR: Character received (executes in <5μs)
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    // Send to stream buffer (lock-free, ISR-safe)
    xStreamBufferSendFromISR(uart_stream_buffer, &rx_char, 1,
                              &xHigherPriorityTaskWoken);

    // Wake UART_Task if it was blocked
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);

    // Re-arm for next character
    HAL_UART_Receive_IT(&huart2, &rx_char, 1);
}

// UART_Task: Wait for characters (TRUE blocking - yields CPU)
void uart_task_handler(void *parameters) {
    char buffer[32];
    while (1) {
        // Block until character available (CPU sleeps!)
        size_t bytes = xStreamBufferReceive(uart_stream_buffer,
                                             buffer, sizeof(buffer),
                                             portMAX_DELAY);
        if (bytes > 0) {
            // Process received characters
            cmd_handler_process(buffer, bytes);
        }

        watchdog_feed(wd_uart_id);  // Prove task is alive
    }
}
```

**Why Stream Buffer Instead of Queue?**

| Feature | Stream Buffer | Queue |
|---------|---------------|-------|
| Data type | Raw bytes (char stream) | Structured items (fixed size) |
| Overhead | Low (no item copy) | Higher (copies entire items) |
| Use case | UART RX, byte streams | Commands, events, messages |
| Blocking | TRUE (yields CPU) | TRUE (yields CPU) |

**Result:** Zero CPU wasted on polling! UART_Task enters BLOCKED state and only wakes when ISR receives a character.

## Watchdog System (Deadlock Detection)

**The Problem:** Tasks can hang or deadlock with no visibility.

**Example Scenario:**
```c
// Task accidentally enters infinite loop
void buggy_task(void *params) {
    while (1) {
        if (some_condition) {
            // Oops! Forgot to update some_condition
            // Task stuck here forever!
        }
        vTaskDelay(pdMS_TO_TICKS(1000));  // Never reached
    }
}
```

In a system without monitoring, this bug goes unnoticed until the entire system appears "frozen."

**My Solution:**

```c
// watchdog.h
typedef uint8_t watchdog_id_t;

// Initialize watchdog (call once in main)
void watchdog_init(void);

// Register task for monitoring (call during task init)
watchdog_id_t watchdog_register(const char *task_name, uint32_t timeout_ms);

// Feed watchdog (call regularly in task loop)
void watchdog_feed(watchdog_id_t id);
```

**Usage:**

```c
void uart_task_handler(void *parameters) {
    watchdog_id_t wd_id = watchdog_register("UART_Task", 5000);  // 5s timeout

    while (1) {
        // Do work...
        process_uart_data();

        // Prove task is alive
        watchdog_feed(wd_id);  // Must call every <5s

        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}
```

**Watchdog Task (Priority 4 - Highest):**

```c
void watchdog_task_handler(void *parameters) {
    while (1) {
        uint32_t now = xTaskGetTickCount();

        for (int i = 0; i < MAX_TASKS; i++) {
            if (watchdog_tasks[i].registered) {
                uint32_t elapsed = now - watchdog_tasks[i].last_feed;

                if (elapsed > watchdog_tasks[i].timeout_ms) {
                    // ALERT: Task hung!
                    char alert[128];
                    snprintf(alert, sizeof(alert),
                        "\r\n*** WATCHDOG ALERT ***\r\n"
                        "Task: %s\r\nLast feed: %lu ms ago\r\n",
                        watchdog_tasks[i].task_name, elapsed);
                    print_message(alert);
                }
            }
        }

        vTaskDelay(pdMS_TO_TICKS(1000));  // Check every 1 second
    }
}
```

**Real-World Example (From Testing):**

```
[CMD] User selected Pattern 2
[LED] Pattern 2: Different frequencies

(Disconnected UART cable to simulate hardware failure)

*** WATCHDOG ALERT ***
Task: UART_Task
Last feed: 5234 ms ago
Timeout: 5000 ms
Status: HUNG/DEADLOCK SUSPECTED
***********************
```

**Impact:** Saved hours of debugging time during development!

## Memory Usage Analysis

```bash
arm-none-eabi-size led_control.elf
```

**Output:**
```
   text    data     bss     dec     hex filename
  42580     124   66192  108896   1a9a0 led_control.elf
```

| Component | Size | Notes |
|-----------|------|-------|
| **Flash (.text + .data)** | 42.7 KB / 1 MB | 4.2% usage |
| **BSS (globals + stacks)** | 64.6 KB / 192 KB | 33.6% usage |
| **Heap (FreeRTOS)** | 75 KB | Configured in FreeRTOSConfig.h |
| **Total RAM** | ~140 KB | 73% utilization |

**Heap Breakdown:**
- Print queue: 10 messages × 512 bytes = 5.1 KB
- Stream buffer: 128 bytes
- Task stacks: ~6.7 KB (5 tasks × ~300 words avg)
- FreeRTOS kernel: ~8 KB
- Free heap: ~55 KB (73% available for expansion)

**Comparison to Part 1 (Bare Metal):**

| Metric | Bare Metal | FreeRTOS | Increase |
|--------|------------|----------|----------|
| Flash | 2.06 KB | 42.7 KB | **20x** |
| RAM | 24 bytes | 140 KB | **5833x** |

**Why the huge increase?**
- FreeRTOS kernel code (~18 KB flash)
- HAL libraries (~15 KB flash)
- Task stacks (each task needs dedicated stack)
- Print queue buffers (5.1 KB)
- Heap for dynamic memory (75 KB)

**Is it worth it?** Absolutely! The scalability, maintainability, and features justify the memory cost for any non-trivial application.

## Challenges Faced & Solutions

### 1. Heap Exhaustion Crash

**Problem:** System crashed on boot with "Hard Fault" error.

**Root Cause:** Heap size (configTOTAL_HEAP_SIZE) set too large:
```c
#define configTOTAL_HEAP_SIZE  ( ( size_t ) ( 100 * 1024 ) )  // 100 KB
```
But BSS (global variables + stacks) was already 66 KB, leaving only 126 KB total RAM.
100 KB heap + 66 KB BSS = 166 KB > 192 KB available → **crash!**

**Solution:** Reduced heap to 75 KB:
```c
#define configTOTAL_HEAP_SIZE  ( ( size_t ) ( 75 * 1024 ) )  // 75 KB
```

**Memory calculation:**
```
Total RAM: 192 KB
BSS usage: 66 KB
Heap: 75 KB
──────────────────
Total: 141 KB ✅ (51 KB margin for safety)
```

**Lesson:** Always account for BSS when sizing FreeRTOS heap!

### 2. Stack Overflow in Print Task

**Problem:** System crashed after printing long messages.

**Root Cause:** Print task stack too small (128 words = 512 bytes):
```c
xTaskCreate(print_task_handler, "Print_Task", 128, ...);  // TOO SMALL!
```

But `snprintf()` inside print task used large stack buffers:
```c
char alert[256];  // 256 bytes on stack
snprintf(alert, sizeof(alert), "Long message...");  // Overflow!
```

**Solution:** Increased stack to 512 words (2048 bytes):
```c
xTaskCreate(print_task_handler, "Print_Task", 512, ...);  // ✅ Safe
```

**How to debug:** Enable FreeRTOS stack overflow detection:
```c
// FreeRTOSConfig.h
#define configCHECK_FOR_STACK_OVERFLOW  2  // Method 2 (most thorough)

// Hook function (called when overflow detected)
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) {
    printf("Stack overflow in task: %s\r\n", pcTaskName);
    while(1);  // Halt for debugging
}
```

### 3. UART Transmission Corruption

**Problem:** Garbled UART output when multiple tasks printed simultaneously:
```
Expected: "[LED] Pattern 1\r\n[CMD] User selected 1\r\n"
Actual:   "[LED] Pa[CMD] Usertterselected 11\r\n"
```

**Root Cause:** Multiple tasks called `HAL_UART_Transmit()` concurrently, interleaving bytes.

**Failed Solution #1:** Mutex protection
```c
// This fixes corruption, but...
xSemaphoreTake(uart_mutex, portMAX_DELAY);
HAL_UART_Transmit(&huart2, buffer, len, 100);  // Task blocks here!
xSemaphoreGive(uart_mutex);
```
- Task blocks for entire transmission duration (~1ms per 10 bytes @ 115200 baud)
- Priority inversion possible
- Mutex boilerplate in 10+ places

**Working Solution #2:** Dedicated print task (described earlier)
- Non-blocking for callers
- No mutex needed
- Clean separation of concerns

## What I Learned

**FreeRTOS Strengths:**
- ✅ **Scalability** - Adding new tasks is trivial
- ✅ **Pre-emptive scheduling** - High-priority tasks run immediately
- ✅ **Blocking primitives** - Tasks yield CPU instead of busy-waiting
- ✅ **Power efficiency** - Idle task enters WFI sleep mode
- ✅ **Rich ecosystem** - Queues, semaphores, timers, event groups all built-in

**FreeRTOS Complexity:**
- ⚠️ **Memory overhead** - 20x flash, 5800x RAM vs. bare-metal
- ⚠️ **Configuration** - FreeRTOSConfig.h has 50+ options
- ⚠️ **Debugging** - Stack overflows, priority inversions, deadlocks require careful analysis
- ⚠️ **Learning curve** - Understanding scheduler, tick rate, priorities takes time

**Key Takeaway:** For any application beyond 3-4 concurrent tasks, **FreeRTOS is essential**. The initial complexity pays off in maintainability and scalability.

## Next Steps

In [Part 3][part3], I shift focus to **wireless communication** with an ESP8266 Wi-Fi web server:

- RESTful HTTP API for LED control
- Responsive web interface
- Client tracking and request history
- All without the STM32 (ESP8266 standalone)

Then in [Part 4][part4], I integrate everything:
- STM32 (FreeRTOS + print task + watchdog)
- ESP8266 (Wi-Fi web server)
- UART bridge connecting them
- Error handling and collision prevention

## Code Repository

**Full source code:** [github.com/sharan-naribole/rtos-led-control-uart-menu](https://github.com/sharan-naribole/rtos-led-control-uart-menu)

Key files:
- `src/main.c` - Application entry point
- `src/print_task.c` - Dedicated UART TX task
- `src/uart_task.c` - UART RX with stream buffer
- `src/cmd_handler.c` - Menu state machine
- `src/led_effects.c` - Software timers for LED patterns
- `src/watchdog.c` - Task monitoring system
- `Architecture.md` - Detailed technical documentation (33 KB!)
- `WATCHDOG_USAGE.md` - Watchdog API guide

---

**← Previous:** [Part 1: Multi-Task LED Blinker Without RTOS](/posts/2025/09/02/stm32-led-blinker-bare-metal)

**Next in Series →** [Part 3: ESP8266 Wi-Fi Web Server](/posts/2025/12/07/esp8266-wifi-web-server)

[part1]: https://sharan-naribole.github.io/posts/2025/09/02/stm32-led-blinker-bare-metal
[part2]: https://sharan-naribole.github.io/posts/2025/12/03/stm32-freertos-uart-menu
[part3]: https://sharan-naribole.github.io/posts/2025/12/07/esp8266-wifi-web-server
[part4]: https://sharan-naribole.github.io/posts/2025/12/14/stm32-esp8266-iot-led-controller
