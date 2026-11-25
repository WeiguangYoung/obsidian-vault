---
{"dg-publish":true,"permalink":"/knowledge/embedded/09-FreeRTOS入门/","title":"FreeRTOS 入门：从实际工程的第一段代码读起","tags":["FreeRTOS","ESP32-C3","RTOS","任务调度","任务间通信","嵌入式","ESP-IDF","学习路线"],"dg-note-properties":{"title":"FreeRTOS 入门：从实际工程的第一段代码读起","date":"2026-11-25","tags":["FreeRTOS","ESP32-C3","RTOS","任务调度","任务间通信","嵌入式","ESP-IDF","学习路线"]}}
---


# 🧵 FreeRTOS 入门：从实际工程的第一段代码读起

> 不讲抽象的 RTOS 理论，而是**从一段真实在跑的代码开始读**，把 FreeRTOS 的核心概念一个个挖出来。
> 起点是 `esp32c3-demo` 的 `main/main.c` —— 一个 UART1 探针程序，60 行核心逻辑，却用到了
> 任务创建、优先级、栈分配、Tick、延时、日志、错误处理这七件套。
> 读完这篇，你应该能独立判断「这个场景该开任务还是轮询」「栈给多大」「用什么方式传数据」。

返回 [[首页\|🏠 首页]] · 相关：[[knowledge/embedded/08-ESP32C3烧录与裸机固件实战\|ESP32-C3 烧录与裸机固件实战]] · [[knowledge/embedded/07-ESP32C3做CAN与UDS对端节点\|ESP32-C3 做 CAN 与 UDS 对端节点]]

---

## 📑 本页导航

| 章 | 主题 |
|:--|:--|
| 一 | 为什么需要 RTOS —— 从裸机主循环的困境说起 |
| 二 | 起点代码：esp32c3-demo 的 main.c |
| 三 | 逐行拆解：这段代码里的 FreeRTOS |
| 四 | 任务：创建、优先级、栈 |
| 五 | 时间：Tick 与两种延时 |
| 六 | 任务间通信（入门只需掌握两个） |
| 七 | ESP-IDF 的 FreeRTOS 与原生差异 |
| 八 | 从这段代码能学到的 7 个模式 |
| 九 | 常见坑 |
| 十 | 后续学习路线 |

---

## 一、为什么需要 RTOS —— 从裸机主循环的困境说起

你在 [[knowledge/embedded/08-ESP32C3烧录与裸机固件实战\|ESP32-C3 裸机固件实战]] 里写过这样的主循环：

```c
while (1) {
    if (uart_has_data()) {        /* 收数据 */
        handle(uart_read());
    }
    if (time_to_blink()) {        /* 点灯 */
        toggle_led();
    }
    if (time_to_report()) {       /* 上报 */
        send_status();
    }
}
```

**能跑，但三个问题绕不过去**：

| 问题 | 表现 | 根本原因 |
|:--|:--|:--|
| **响应延迟不可控** | 某次 `handle()` 耗时 200ms，点灯就晚 200ms | 单线程，谁在跑谁独占 CPU |
| **阻塞调用杀死一切** | 调一次阻塞式 `uart_read()` 等数据，灯就不闪了 | 没有「让出 CPU」的机制 |
| **逻辑越加越乱** | 四个功能之后，主循环变成一堆 `if` 嵌套 | 功能之间没有隔离边界 |

**RTOS 给的就是这三件事**：

1. **多个执行流**（任务），各自独立写，互不干扰
2. **抢占式调度**：高优先级任务就绪就能立刻拿到 CPU
3. **阻塞语义**：任务「等数据」时主动让出 CPU，不空转

> **一句话**：裸机主循环 = 一个人干所有活；RTOS = 几个人分工 + 一个调度员。

---

## 二、起点代码：esp32c3-demo 的 main.c

`esp32c3-demo/main/main.c` 是一个 **UART1 逻辑串口探针**。它要解决的问题很具体：

> 这块板唯一的串口是 CH343 → UART0（GPIO20/21），被控制台日志独占。
> 想做「第二路串口」，只能用 UART1 + GPIO Matrix 重映射到空闲脚。

**核心逻辑只有 60 行**：

```c
/* ---- 引脚映射 ---- */
#define U1_PORT         UART_NUM_1
#define U1_TX_GPIO      4
#define U1_RX_GPIO      5
#define U1_BAUD         115200
#define U1_RX_BUF       2048
#define U1_TX_BUF       0        /* 0 = 不建 TX 环形缓冲，直接阻塞写 */
#define U1_HB_MS        5000     /* 心跳周期 */

void app_main(void)
{
    vTaskDelay(pdMS_TO_TICKS(300));      /* 等 UART0 控制台稳定 */

    ESP_LOGI(TAG, "=== esp32c3 uart1 probe ===");
    print_chip_info();

    ESP_ERROR_CHECK(uart1_init());

#if ENABLE_LOOPBACK_TEST
    bool lb = uart1_internal_loopback_test();
    ESP_LOGI(TAG, "loopback self-test: %s", lb ? "PASS" : "FAIL");
#endif

    xTaskCreate(uart1_echo_task, "u1_echo", 3072, NULL, 5, NULL);

    const char *banner = "\r\n=== UART1 ready on GPIO4(TX)/GPIO5(RX) ===\r\n";
    uart_write_bytes(U1_PORT, banner, strlen(banner));

    ESP_LOGI(TAG, "echo task started");
}
```

**工作任务是这个**：

```c
static void uart1_echo_task(void *arg)
{
    uint8_t buf[128];
    uint32_t hb = 0, ping = 0;
    TickType_t last_hb   = xTaskGetTickCount();
    TickType_t last_ping = xTaskGetTickCount();

    while (1) {
        /* 收：有数据就回显 */
        int n = uart_read_bytes(U1_PORT, buf, sizeof(buf), pdMS_TO_TICKS(50));
        if (n > 0) {
            uart_write_bytes(U1_PORT, buf, n);
            if (strstr((char *)buf, "PING")) {
                ESP_LOGI(TAG, "duplex OK (GPIO%d -> GPIO%d): rx %d bytes",
                         U1_TX_GPIO, U1_RX_GPIO, n);
            }
        }

        /* 发：每 2 秒一个 PING */
        if ((xTaskGetTickCount() - last_ping) >= pdMS_TO_TICKS(2000)) {
            last_ping = xTaskGetTickCount();
            char line[32];
            int m = snprintf(line, sizeof(line), "PING %lu\r\n", (unsigned long)++ping);
            uart_write_bytes(U1_PORT, line, m);
        }

        /* 心跳：证明任务没死 */
        if ((xTaskGetTickCount() - last_hb) >= pdMS_TO_TICKS(U1_HB_MS)) {
            last_hb = xTaskGetTickCount();
            ESP_LOGI(TAG, "alive %lu (ping %lu)", (unsigned long)++hb, (unsigned long)ping);
        }
    }
}
```

**为什么这段代码适合当入门教材**：它短，但完整包含了「初始化 → 建任务 → 任务内三件事并发（收、发、心跳）」的骨架。**你要写的任何 RTOS 程序，形态都和它一样。**

---

## 三、逐行拆解：这段代码里的 FreeRTOS

把 RTOS 相关的行挑出来，一共 7 处：

| # | 代码 | 用到什么 |
|:--:|:--|:--|
| 1 | `void app_main(void)` | ESP-IDF 的入口约定（不是 `main`） |
| 2 | `vTaskDelay(pdMS_TO_TICKS(300))` | 延时 + 毫秒转 Tick |
| 3 | `xTaskCreate(uart1_echo_task, "u1_echo", 3072, NULL, 5, NULL)` | 建任务：函数、名字、栈、参数、优先级 |
| 4 | `TickType_t last_ping = xTaskGetTickCount()` | Tick 计数器 |
| 5 | `uart_read_bytes(..., pdMS_TO_TICKS(50))` | **带超时的阻塞读**（让出 CPU 的关键） |
| 6 | `xTaskGetTickCount() - last_ping >= pdMS_TO_TICKS(2000)` | 非阻塞的周期判断 |
| 7 | `ESP_LOGI(TAG, ...)` | ESP-IDF 日志（内部带互斥锁） |

**这七行就是 90% 的日常用法。** 下面逐个展开。

---

## 四、任务：创建、优先级、栈

### 4.1 `xTaskCreate` 的五个参数

```c
xTaskCreate(
    uart1_echo_task,   // ① 任务函数：void (*)(void *)
    "u1_echo",         // ② 名字：仅用于调试（看任务列表时显示）
    3072,              // ③ 栈大小：单位是「字」不是「字节」！
    NULL,              // ④ 传给任务的参数
    5,                 // ⑤ 优先级：数字越大越优先
    NULL               // ⑥ 任务句柄输出，不需要就传 NULL
);
```

**③ 栈大小是最容易搞错的一处**：

| 平台 | `xTaskCreate` 第 3 个参数的单位 | 3072 实际是多少 |
|:--|:--|:--|
| ESP-IDF（ESP32 系列） | **字节**（ESP-IDF 改过，`StackType_t` = 1 字节） | 3072 字节 |
| 原生 FreeRTOS（多数 MCU） | **字**（`StackType_t` = 4 字节） | **12 KB** |

> 📌 ESP-IDF 把 `StackType_t` 定义成了 `uint8_t`，所以这里的 3072 就是字节。但你移植到 STM32 原生 FreeRTOS 时，同样的 3072 会变成 12 KB——**这是移植时最常见的栈浪费**。

**这个任务为什么给 3072 字节**：

```c
uint8_t buf[128];              // 128 字节局部数组
char line[32];                 // 32 字节
// + snprintf 内部开销（浮点/整数格式化用栈较多）
// + ESP_LOGI 的格式化缓冲
// + 函数调用栈（uart_read_bytes 内部还有几层）
```

粗算 128 + 32 = 160 字节的数据，但 `snprintf` 和 `ESP_LOGI` 的格式化路径会吃掉一千多字节。**所以「有格式化输出 + 局部缓冲」的任务，2 KB 起步，3 KB 更稳。**

### 4.2 优先级怎么定

这段代码只有两个任务（`app_main` 本身算一个临时任务，跑完就退出）：

| 任务 | 优先级 | 理由 |
|:--|:--:|:--|
| `app_main` | 1（ESP-IDF 默认） | 只做初始化，做完就结束 |
| `u1_echo` | **5** | 串口数据处理，需要及时响应 |

**ESP-IDF 的优先级范围**：`0` ~ `configMAX_PRIORITIES-1`（默认 24，即 0~24）。

**实用原则**（不需要背理论）：

| 任务类型 | 建议优先级 |
|:--|:--|
| 空闲后台任务（日志上传、统计） | 1 ~ 2 |
| 普通业务（本代码的串口回显） | 5 ~ 8 |
| 实时响应（电机控制、CAN 收发） | 10 ~ 15 |
| 看门狗喂狗、系统关键 | 20+ |

> ⚠️ **别乱用高优先级**：优先级高于所有任务的死循环会把低优先级任务饿死（连 IDLE 任务都跑不到，触发出看门狗复位）。

### 4.3 任务函数必须是死循环

```c
static void uart1_echo_task(void *arg)
{
    while (1) {        /* ← 必须有这层 */
        ...
    }
}
```

**任务函数如果返回了会怎样**：FreeRTOS 会调用 `vTaskDelete(NULL)` 自杀。这在某些配置下触发断言，日志里看到：

```
abort() was called at PC 0x...
Backtrace: ...
```

**要么 `while(1)` 死循环，要么显式 `vTaskDelete(NULL)`**，不要靠隐式删除。

---

## 五、时间：Tick 与两种延时

### 5.1 Tick 是什么

FreeRTOS 靠一个硬件定时器周期性中断来驱动调度，每次中断叫一个 **tick**。

**ESP-IDF 默认 tick 频率 = 100 Hz**，即 **1 tick = 10 ms**。

```c
#define portTICK_PERIOD_MS   (1000 / configTICK_RATE_HZ)   // 默认 = 10
```

**影响**：

| tick 频率 | 1 tick | 时间精度 | 代价 |
|:--|--:|:--|:--|
| 100 Hz（默认） | 10 ms | ±10ms | 中断开销小 |
| 1000 Hz | 1 ms | ±1ms | 中断开销大，但适合 UDS/电机等毫秒级场景 |

改法（`sdkconfig.defaults`）：
```ini
CONFIG_FREERTOS_HZ=1000
```

> 这就是 [[knowledge/embedded/07-ESP32C3做CAN与UDS对端节点\|CAN/UDS 对端节点]] 那篇里要求把 tick 改成 1000 Hz 的原因 —— UDS 的 `STmin` 最小是 1 ms 量级，10 ms 的粒度根本表达不了。

### 5.2 `pdMS_TO_TICKS` 为什么必须用

```c
vTaskDelay(pdMS_TO_TICKS(300));   // ✅ 300ms → 30 ticks（100Hz 时）
vTaskDelay(300);                  // ❌ 300 ticks = 3000ms = 3 秒！
```

**`vTaskDelay` 的参数单位是 tick，不是毫秒。** 直接传 300 会等 3 秒。

**而 `pdMS_TO_TICKS` 的好处是**：改了 tick 频率，代码不用动。100Hz 下 300 → 30，1000Hz 下 300 → 300，行为都是 300ms。

### 5.3 两种延时：`vTaskDelay` vs `vTaskDelayUntil`

这段代码用的是**自己记时间戳**的方式：

```c
TickType_t last_ping = xTaskGetTickCount();
...
if ((xTaskGetTickCount() - last_ping) >= pdMS_TO_TICKS(2000)) {
    last_ping = xTaskGetTickCount();
    /* 干活 */
}
```

**为什么不用 `vTaskDelay(2000)`**：

```c
while (1) {
    do_work();              // 耗时假设 100ms
    vTaskDelay(pdMS_TO_TICKS(2000));
}
```

实际周期 = 2000 + 100 = **2100ms**，而且 `do_work()` 的耗时一变，周期就跟着漂。

**改成 `vTaskDelayUntil` 就能固定周期**：

```c
TickType_t last = xTaskGetTickCount();
while (1) {
    do_work();
    vTaskDelayUntil(&last, pdMS_TO_TICKS(2000));   // 严格 2000ms 周期
}
```

**三种方式对比**：

| 方式 | 周期精度 | 适用 |
|:--|:--|:--|
| `vTaskDelay(n)` | 周期 = n + 处理耗时（漂移） | 不需要固定周期的场景 |
| `vTaskDelayUntil(&last, n)` | 严格 n（自动补偿处理耗时） | 周期性采样、定时上报 |
| 时间戳比较（本代码） | 精度 = 循环轮询间隔 | **一个任务里要同时处理多个不同周期的事** |

> **本代码为什么选第三种**：它在同一个循环里要处理「收数据（50ms 超时）」「发 PING（2s）」「心跳（5s）」三件不同周期的事。如果用 `vTaskDelayUntil`，只能有一个主周期，另外两件就得拆成独立任务。

**这也是一个重要的设计取舍**：

```
方案 A：一个任务 + 时间戳判断  → 省栈、省调度开销，但逻辑耦合
方案 B：三个任务各管一件事    → 逻辑清晰，但每个任务 2~3KB 栈
```

**小工程选 A，复杂的选 B。**

### 5.4 `xTaskGetTickCount() - last` 为什么要相减

```c
if ((xTaskGetTickCount() - last_ping) >= pdMS_TO_TICKS(2000))
```

**不要写成 `xTaskGetTickCount() >= last_ping + 2000`** —— 因为 `TickType_t` 是**无符号整数，会溢出回绕**。

- 32 位 tick @ 1000Hz → 约 49.7 天回绕一次
- 回绕时 `now` 突然变成很小的值，`now >= last + 2000` 就永远不成立了

**相减的写法在回绕时依然正确**（无符号运算的自洽性），这是 FreeRTOS 的经典惯用法。

---

## 六、任务间通信（入门只需掌握两个）

这段代码只有**一个任务**，所以没用到通信机制。但只要你开第二个任务，立刻就需要。

### 6.1 队列（Queue）：传数据

**场景**：串口收到的数据要交给协议解析任务。

```c
/* 创建：长度 16，每项是 uint8_t */
QueueHandle_t q = xQueueCreate(16, sizeof(uint8_t));

/* 发送方（串口任务） */
uint8_t byte = 0x55;
xQueueSend(q, &byte, pdMS_TO_TICKS(10));    /* 队列满时最多等 10ms */

/* 接收方（解析任务） */
uint8_t got;
if (xQueueReceive(q, &got, portMAX_DELAY) == pdTRUE) {
    /* 拿到一个字节，队列空时会一直阻塞在这里（不占 CPU） */
}
```

**关键点**：

| 参数 | 含义 |
|:--|:--|
| 创建时的第 1 个参数 | **队列能存几项**（不是字节数） |
| 创建时的第 2 个参数 | **每项多少字节**（`sizeof(项类型)`） |
| 发送/接收的超时 | `0` = 不等；`portMAX_DELAY` = 永久等 |

**队列是「拷贝」语义**：`xQueueSend(q, &byte, ...)` 拷贝的是 `byte` 的值，不是指针。所以局部变量可以直接传。

> ⚠️ 但如果项是指针（如 `char *`），拷贝的是指针，指向的内存必须保证在接收方用完之前有效 —— 这就是「队列传指针」的坑。

### 6.2 信号量（Semaphore）：传事件

**场景**：GPIO 中断里收到一帧数据，通知任务去处理。

```c
SemaphoreHandle_t sem = xSemaphoreCreateBinary();

/* 中断里（必须用 FromISR 版本） */
BaseType_t woken = pdFALSE;
xSemaphoreGiveFromISR(sem, &woken);
portYIELD_FROM_ISR(woken);

/* 任务里 */
if (xSemaphoreTake(sem, pdMS_TO_TICKS(1000)) == pdTRUE) {
    /* 拿到信号，处理数据 */
}
```

**队列 vs 信号量怎么选**：

| | 队列 | 信号量 |
|:--|:--|:--|
| 传什么 | **数据**（有内容） | **事件**（只表示「发生了」） |
| 能存几个 | 多个（先进先出） | 二值（0/1）或计数 |
| 典型场景 | 串口字节流、CAN 帧 | 中断通知、资源占用 |

### 6.3 剩下的先别学

入门阶段**只学队列和信号量**就够了。以下这些等到实际遇到问题再看：

| 机制 | 什么时候才需要 |
|:--|:--|
| 互斥锁（Mutex） | 多个任务**同时写**同一个共享变量/外设 |
| 事件组（Event Group） | 要等多个条件**全部满足**才开始 |
| 任务通知（Task Notify） | 队列/信号量的轻量替代（省内存） |
| 流缓冲（Stream Buffer） | 高效传**字节流**（比队列省） |
| 消息缓冲（Message Buffer） | 传**变长消息** |

> **一条实用建议**：先只用队列 + 信号量把功能写出来。真遇到性能问题或内存紧张时再换 —— **过早优化通信机制是新手最常见的过度设计**。

---

## 七、ESP-IDF 的 FreeRTOS 与原生差异

如果你以后要从 ESP32 换到 STM32 原生 FreeRTOS，这几处会踩坑：

| 差异点 | ESP-IDF | 原生 FreeRTOS |
|:--|:--|:--|
| **入口函数** | `app_main()` | `main()`（自己建初始任务） |
| **栈单位** | 字节 | 字（4 字节） |
| **调度启动** | 框架自动 `vTaskStartScheduler()` | 手动调 |
| **默认 tick** | 100 Hz | `configTICK_RATE_HZ` 自定，常 1000 |
| **空闲任务钩子** | 有专门的 idle task 配置 | `configUSE_IDLE_HOOK` |
| **日志** | `ESP_LOGI` 等你直接用 | 自己实现或不用 |
| **双核支持** | `xTaskCreatePinnedToCore` | 单核无此函数 |

### 7.1 `app_main` 不是 `main`

```c
void app_main(void)     // ESP-IDF 的入口
{
    /* 这里已经跑在 FreeRTOS 里了 */
    /* 可以直接调 vTaskDelay、xTaskCreate */
}
```

**它本身是一个任务**（不是主线程），默认优先级 1，栈 3584 字节。**`app_main` 返回后任务就被删除了** —— 所以里面 `xTaskCreate` 建的任务能继续跑，但你自己不能在 `app_main` 里依赖返回后的行为。

### 7.2 错误处理：`ESP_ERROR_CHECK` vs `ESP_RETURN_ON_ERROR`

这段代码用了两种：

```c
/* ① 致命错误：直接 abort + 重启 */
ESP_ERROR_CHECK(uart1_init());

/* ② 可恢复错误：打日志 + 返回错误码 */
ESP_RETURN_ON_ERROR(uart_param_config(U1_PORT, &cfg), TAG, "param_config failed");
```

| 宏 | 失败时行为 | 用在哪 |
|:--|:--|:--|
| `ESP_ERROR_CHECK(x)` | 打日志 + `abort()` → 重启 | 初始化阶段，失败就没法继续 |
| `ESP_RETURN_ON_ERROR(x, tag, msg)` | 打日志 + `return err` | 函数内部，把错误往上抛 |

**`ESP_RETURN_ON_ERROR` 本质是个宏**，展开后是：

```c
do {
    esp_err_t err_rc_ = (x);
    if (err_rc_ != ESP_OK) {
        ESP_LOGE(tag, "%s", msg);   /* 还会打文件名和行号 */
        return err_rc_;
    }
} while (0)
```

用 `do { } while(0)` 包起来是为了让它在 `if` 后面当单条语句用（宏的标准技巧）。

---

## 八、从这段代码能学到的 7 个模式

这 7 条可以直接搬到任何 ESP-IDF 项目里：

### 模式 1：入口先等一下控制台

```c
vTaskDelay(pdMS_TO_TICKS(300));   /* 等 UART0 控制台稳定 */
```

**为什么**：ESP32 上电到 `app_main` 执行只需几十毫秒，但 USB-Serial-JTAG / CH343 的枚举和主机端终端连接可能要更久。不等的话，**开头几行日志会丢**。

### 模式 2：初始化函数返回 `esp_err_t`

```c
static esp_err_t uart1_init(void)
{
    ESP_RETURN_ON_ERROR(uart_param_config(...), TAG, "param_config failed");
    ESP_RETURN_ON_ERROR(uart_set_pin(...), TAG, "set_pin failed");
    ESP_RETURN_ON_ERROR(uart_driver_install(...), TAG, "driver_install failed");
    return ESP_OK;
}
```

**规律**：`esp_err_t` 返回 + `ESP_RETURN_ON_ERROR` 串联，调用方一行 `ESP_ERROR_CHECK()` 检查。**别在初始化函数里 `abort()`**——让调用方决定是致命错误还是可降级。

### 模式 3：任务只做一件事，但循环里可以有多件事

```c
static void uart1_echo_task(void *arg)
{
    while (1) {
        /* 收（50ms 超时，没数据也让出 CPU） */
        /* 发（2s 一次） */
        /* 心跳（5s 一次） */
    }
}
```

**关键在「收」那一步的 50ms 超时**：

```c
int n = uart_read_bytes(U1_PORT, buf, sizeof(buf), pdMS_TO_TICKS(50));
```

如果写 `portMAX_DELAY`，没数据时任务就永久阻塞，PING 和心跳永远发不出去。**超时参数是让多个周期任务共存的关键。**

### 模式 4：用「最后执行时刻」而不是「剩余时间」

```c
TickType_t last_ping = xTaskGetTickCount();
...
if ((xTaskGetTickCount() - last_ping) >= pdMS_TO_TICKS(2000)) {
    last_ping = xTaskGetTickCount();   /* 更新基准点 */
    /* 干活 */
}
```

**别用计数器递减**（`countdown--`），因为循环执行频率不固定，递减就不准。**记录绝对时刻 + 相减比较**，永远准。

### 模式 5：心跳日志证明任务没死

```c
if ((xTaskGetTickCount() - last_hb) >= pdMS_TO_TICKS(U1_HB_MS)) {
    last_hb = xTaskGetTickCount();
    ESP_LOGI(TAG, "alive %lu (ping %lu)", (unsigned long)++hb, (unsigned long)ping);
}
```

**为什么重要**：嵌入式调试最常见的困惑是「程序卡住了还是没输出」。**定期心跳能立刻区分**：

| 现象 | 含义 |
|:--|:--|
| 心跳在涨，但没别的输出 | 任务活着，在忙别的事 |
| 心跳不涨了 | 任务卡死 / 死循环 / 被更高优先级饿死 |
| 心跳次数暴涨 | 系统在重启（每次重启从 0 开始） |

> 这条经验在 [[knowledge/embedded/08-ESP32C3烧录与裸机固件实战\|裸机实战]] 里也用过：「看起来死机但 tick 在涨」= 阻塞在主循环，不是 CPU 挂了。

### 模式 6：自测写在启动流程里，用编译开关控制

```c
#if ENABLE_LOOPBACK_TEST
    bool lb = uart1_internal_loopback_test();
    ESP_LOGI(TAG, "loopback self-test: %s", lb ? "PASS" : "FAIL");
#endif
```

**好处**：量产固件把 `ENABLE_LOOPBACK_TEST` 设 0，一行代码控制，不用改逻辑。

### 模式 7：先跑通过自环，再依赖外部接线

```c
static bool uart1_internal_loopback_test(void)
{
    /* 开片内回环：TX 在芯片内部直接接回 RX，不经过引脚 */
    uart_set_loop_back(U1_PORT, true);
    uart_write_bytes(U1_PORT, pattern, len);
    int rd = uart_read_bytes(U1_PORT, rx, len, pdMS_TO_TICKS(500));
    uart_set_loop_back(U1_PORT, false);
    /* ... */
}
```

**两级验证的分工**：

| 测试 | 证明什么 |
|:--|:--|
| 内部回环 | **外设本身 + 驱动链路**正常 |
| 外部飞线短接 GPIO4/GPIO5 | **引脚映射（GPIO Matrix）**也正确 |

**先排除「外设坏了」，再排除「接线错了」，两者不能混在一起查。**

---

## 九、常见坑

| # | 坑 | 现象 | 解法 |
|:--:|:--|:--|:--|
| 1 | `vTaskDelay(300)` 当毫秒用 | 等了 3 秒 | 用 `pdMS_TO_TICKS(300)` |
| 2 | `xTaskCreate` 栈按字算 | ESP32 上栈不够崩，STM32 上浪费 4 倍 | ESP-IDF 是字节，原生是字，移植时重算 |
| 3 | 任务函数没死循环 | `abort()` / 反复重启 | `while(1)` 或 `vTaskDelete(NULL)` |
| 4 | 高优先级任务死循环 | 低优先级任务饿死，看门狗复位 | 循环里必须有阻塞点（`vTaskDelay` / 带超时的读） |
| 5 | `xTaskGetTickCount() >= last + N` | 49 天后莫名失效 | 用 `(now - last) >= N` |
| 6 | 在中断里用非 ISR 版 API | 崩溃 / 断言 | `xQueueSendFromISR` / `xSemaphoreGiveFromISR` |
| 7 | 栈给太小 + `snprintf` | 随机崩溃，报 `Stack canary watchpoint triggered` | 有格式化输出的任务 ≥ 2KB（建议 3KB） |
| 8 | 多个任务同时 `printf` | 日志串行错乱 | ESP-IDF 的 `ESP_LOGI` 自带锁；裸 `printf` 要自己加互斥 |
| 9 | `app_main` 里做大循环 | 阻塞了系统任务（如 WiFi） | `app_main` 只做初始化，逻辑放独立任务 |
| 10 | 队列项传指针 | 数据错乱 / 野指针 | 传值，或保证内存生命周期 |

### 栈不够怎么查

ESP-IDF 有现成的运行时统计：

```c
/* 查某个任务栈剩余最小值（字节） */
UBaseType_t hw = uxTaskGetStackHighWaterMark(NULL);   /* NULL = 当前任务 */
ESP_LOGI(TAG, "stack free: %u bytes", hw * sizeof(StackType_t));
```

**高水位线（High Water Mark）** = 任务生命周期内**栈用得最多时还剩多少**。这个值接近 0 就危险了。

**建议**：开发阶段每 10 秒打一次，跑够业务所有分支后再定最终栈大小（留 30% 余量）。

也可以看整个系统的任务：

```c
/* 需要 CONFIG_FREERTOS_USE_TRACE_FACILITY=y */
char buf[1024];
vTaskList(buf);       /* 列出所有任务：名字、状态、优先级、剩余栈 */
printf("%s\n", buf);
```

**输出格式**：
```
Task Name   State  Prio  Stack  Num
u1_echo     R      5     1024   3      ← Stack 是本任务「最小剩余」，不是总栈
IDLE        R      0     512    2
```

> ⚠️ `vTaskList` 的输出是原始字节/字，不含单位。ESP-IDF 上看到 1024 就是 1024 字节剩余。

---

## 十、后续学习路线

### 10.1 第一步：把这段代码改造成多任务（1~2 天）

**别只读，动手改。** 目标是把 `uart1_echo_task` 拆成三个任务 + 队列：

```
┌──────────────┐   Queue    ┌──────────────┐
│ uart_rx_task │───────────▶│ echo_task    │
│ (优先级 8)    │  字节流     │ (优先级 5)    │
└──────────────┘            └──────────────┘
        │
        │ Semaphore
        ▼
┌──────────────┐
│ heartbeat    │  (优先级 2)
│ (只打日志)    │
└──────────────┘
```

**验收标准**：

- [ ] 三个任务能同时工作，互不阻塞
- [ ] 用 `uxTaskGetStackHighWaterMark` 打出每个任务的栈余量
- [ ] 用 `vTaskList` 打印任务列表，确认优先级和状态符合预期
- [ ] 故意把 `echo_task` 的栈改成 512，观察崩溃现象（认识「栈溢出长什么样」）

### 10.2 第二步：掌握通信与同步（3~5 天）

| 顺序 | 内容 | 练习 |
|:--:|:--|:--|
| 1 | 队列 | 生产者任务每 100ms 发一个结构体，消费者打印 |
| 2 | 二值信号量 | 按键中断 → 任务响应 |
| 3 | 计数信号量 | 3 个资源池，5 个任务抢 |
| 4 | 互斥锁 | 两个任务同时累加一个全局变量，对比加锁/不加锁的结果 |
| 5 | 事件组 | 等「WiFi 连上 + 时间同步」都完成才启动业务 |
| 6 | 任务通知 | 用 Notify 替换队列，对比内存占用 |

**关键认知**：**互斥锁解决的是「数据竞争」，信号量解决的是「事件通知」**。两者不能混用。

### 10.3 第三步：深入调度与内存（3~5 天）

| 内容 | 为什么重要 |
|:--|:--|
| 抢占式 vs 协作式调度 | 理解 `configUSE_PREEMPTION` |
| 优先级反转 | 低优先级持锁 → 高优先级等 → 中优先级插队 |
| 优先级继承（Mutex） | 解决反转的机制 |
| `configMAX_PRIORITIES` | 优先级数量的取舍 |
| 堆方案（heap_1~heap_5） | ESP-IDF 用 heap_4 变体 |
| 静态创建（`xTaskCreateStatic`） | 不用 `malloc` 的嵌入式场景 |

### 10.4 第四步：工程化能力（1~2 周）

| 内容 | 落地方式 |
|:--|:--|
| **任务看门狗（TWDT）** | `esp_task_wdt_add()` + 定期 `esp_task_wdt_reset()` |
| **CPU 占用分析** | `vTaskGetRunTimeStats()`（需 `CONFIG_FREERTOS_GENERATE_RUN_TIME_STATS`） |
| **栈溢出检测** | `CONFIG_FREERTOS_CHECK_STACKOVERFLOW_CANARY` |
| **死锁排查** | 命名每个任务和锁，用 `vTaskList` 看谁在等谁 |
| **低功耗** | Tickless idle（`CONFIG_FREERTOS_USE_TICKLESS_IDLE`） |
| **中断延迟测量** | GPIO 翻转 + 逻辑分析仪 |

### 10.5 第五步：回到你的主线

FreeRTOS 学到这里，可以接回你的项目：

| 方向 | 怎么用 FreeRTOS |
|:--|:--|
| **CAN/UDS 对端节点** | TWAI 接收任务 + UDS 服务任务 + 定时任务（P2/S3 超时）+ 故障注入任务 |
| **ESP32-C3 做产品** | WiFi 事件处理 + 业务任务 + OTA 任务 |
| **多传感器采集** | 每个传感器一个任务，队列汇总到上传任务 |

> 具体到 [[knowledge/embedded/07-ESP32C3做CAN与UDS对端节点\|CAN/UDS 对端节点]]，那篇文档里的任务划分就是这个思路：
> ```c
> xTaskCreate(can_rx_task,   "can_rx",   4096, NULL, 10, NULL);
> xTaskCreate(can_mon_task,  "can_mon",  3072, NULL,  3, NULL);
> xTaskCreate(s3_timer_task, "s3_timer", 2048, NULL,  2, NULL);
> ```
> **优先级递减**：收发（10）> 监控（3）> 超时判断（2）。

### 10.6 推荐资料

| 类型 | 资源 |
|:--|:--|
| **官方文档** | [FreeRTOS 官方手册](https://www.freertos.org/Documentation/RTOS_book.html)（免费 PDF） |
| **ESP-IDF 文档** | `FreeRTOS (IDF)` 章节 —— **优先看这个**，有 ESP32 特有说明 |
| **源码** | `esp-idf/components/freertos/` —— 你板子上跑的就是它 |
| **《Mastering the FreeRTOS Real Time Kernel》** | 官方教程，比手册更易读 |
| **实践** | 本仓库 `esp32c3-demo` 的 `main/` 和 `examples/` |

---

## 📊 一页速查

### 最常用的 10 个 API

| API | 用途 |
|:--|:--|
| `xTaskCreate(fn, name, stack, arg, prio, &handle)` | 建任务 |
| `vTaskDelay(pdMS_TO_TICKS(ms))` | 相对延时 |
| `vTaskDelayUntil(&last, ticks)` | 固定周期 |
| `xTaskGetTickCount()` | 当前 tick |
| `pdMS_TO_TICKS(ms)` | 毫秒 → tick |
| `xQueueCreate(len, size)` | 建队列 |
| `xQueueSend(q, &v, timeout)` / `xQueueReceive(q, &v, timeout)` | 收发队列 |
| `xSemaphoreCreateBinary()` / `xSemaphoreTake/Give` | 信号量 |
| `uxTaskGetStackHighWaterMark(NULL)` | 查栈余量 |
| `vTaskList(buf)` | 列所有任务 |

### 时间换算

```
1 tick = 1000 / configTICK_RATE_HZ 毫秒
默认 100Hz → 1 tick = 10ms
1000Hz     → 1 tick = 1ms
```

### 优先级建议

```
1-2   后台任务（日志、统计）
5-8   普通业务（串口、网络）
10-15 实时响应（电机、CAN）
20+   系统关键（看门狗）
```

---

## 📂 关联文件

- [[knowledge/embedded/08-ESP32C3烧录与裸机固件实战\|ESP32-C3 烧录与裸机固件实战]] — 不用 RTOS 的裸机版本，对比看更清楚 RTOS 解决了什么
- [[knowledge/embedded/07-ESP32C3做CAN与UDS对端节点\|ESP32-C3 做 CAN 与 UDS 对端节点]] — 三任务划分的实战案例
- [[knowledge/embedded/01-编译链接装载与库\|编译、链接、装载与库]] — 栈、堆、链接脚本的底层原理

---

> 🦐 虾管家 · 2026-11-25 编写（以 esp32c3-demo/main/main.c 为起点，只写文档不改代码）