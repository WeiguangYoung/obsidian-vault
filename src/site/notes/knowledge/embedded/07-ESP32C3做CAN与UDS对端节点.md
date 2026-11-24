---
{"dg-publish":true,"permalink":"/knowledge/embedded/07-ESP32C3做CAN与UDS对端节点/","title":"ESP32-C3 做 CAN 与 UDS 对端节点","tags":["ESP32-C3","RISC-V","TWAI","CAN","UDS","ISO-TP","ISO15765","ISO14229","ESP-IDF","诊断","对端节点","嵌入式"],"dg-note-properties":{"title":"ESP32-C3 做 CAN 与 UDS 对端节点","date":"2026-11-24","tags":["ESP32-C3","RISC-V","TWAI","CAN","UDS","ISO-TP","ISO15765","ISO14229","ESP-IDF","诊断","对端节点","嵌入式"]}}
---


# 🔌 ESP32-C3 做 CAN 与 UDS 对端节点

> 做 UDS 开发最难受的一件事：**你手上只有一个"被测端"（自己写的 Bootloader / ECU），却没有"对面那个"**。
> 用 ZLG 盒子加脚本发请求可以，但它只能按你写好的剧本走，没法真的"作为 ECU 应答"，也做不了错误注入。
> ESP32-C3 正好补这个空档：内置 TWAI 控制器（CAN 2.0B），¥17 一块，配一个 3.3V 收发器就能当**可编程的 ECU 对端**。
> 本文从硬件接线、驱动初始化、UDS 服务层、ISO-TP 多帧一直讲到故障注入压测，给出可直接编译的工程结构。

---

## 📑 本页导航

| 章节 | 内容 |
|:--|:--|
| 一、为什么需要自建对端节点 | 与 ZLG / 真 ECU 的对比 |
| 二、硬件选型与接线 | 收发器选择、引脚分配、三个致命坑 |
| 三、开发环境 | ESP-IDF 安装与工程创建 |
| 四、TWAI 驱动层 | 初始化、三种模式、状态监控 |
| 五、UDS 服务层 | 服务分发、肯定/否定响应构造 |
| 六、ISO-TP 多帧 | SF/FF/CF/FC 四种 PCI 与实现 |
| 七、时序参数 | P2 / P2* / S3 三个必做项 |
| 八、联调四步走 | 从单板自测到与真实 ECU 握手 |
| 九、故障注入压测 | 这是自建对端最核心的价值 |
| 十、坑清单 | 十条实战经验 |
| 十一、工程骨架 | 目录结构与 CMakeLists |
| 十二、一页速查 | 表格总结 |

---

## 一、为什么需要自建对端节点

### 三种"对端"方案的对比

| 方案 | 能做 | 不能做 | 成本 |
|:--|:--|:--|:--|
| **真车 ECU** | 最真实 | 不可控、没法注入故障、不能改行为 | 天价 |
| **ZLG 盒子 + 脚本** | 发请求、抓报文 | 不能主动应答、不能改时序、不能造畸形帧 | ¥300+ |
| **ESP32-C3 对端** | 全部可控 | 不是真 ECU（逻辑自己写） | ¥28 |

关键差异在最后一列：**ZLG 是"记录仪"，ESP32-C3 是"演员"**。

你需要的场景：

- 你的 Bootloader 发出 `10 03`（切扩展会话），总得有人回 `50 03 00 32 01 F4`
- 你要测「对端回 NRC 0x78 后延迟 6 秒」时，自己的超时逻辑对不对
- 你要测「连续帧序号错乱」时，自己的 ISO-TP 重组会不会崩
- 你要测「对端完全不响应」时，P2Server 超时是否触发

这些**真 ECU 试不出来**（你没法让博世给你改固件），ZLG 也做不到（它不回请求）。

### 在 UDS 开发中的位置

```
┌──────────────┐        CAN 总线        ┌──────────────┐
│  ESP32-C3    │◄──────────────────────►│  F407        │
│  对端 ECU    │   0x7E0 / 0x7E8       │  被测 Bootloader│
│  (应答方)    │                        │  (请求方)    │
└──────────────┘                        └──────────────┘
        ▲
        │ 监听
┌──────────────┐
│  ZLG 盒子    │   ← 第三只眼，只看不说
└──────────────┘
```

标准诊断 ID 分配（ISO 15765-4，11 位）：

| 方向 | 请求 ID | 响应 ID |
|:--|:--|:--|
| 物理寻址（点对点） | `0x7E0` | `0x7E8` |
| 功能寻址（广播） | `0x7DF` | — |

> `0x7E0 + N` → `0x7E8 + N`，N 是 ECU 序号。单 ECU 场景就固定用 0x7E0/0x7E8。

---

## 二、硬件选型与接线

### 物料清单

| 件 | 型号 | 价格 | 说明 |
|:--|:--|:--:|:--|
| 主控 | ESP32-C3（如 SuperMini / DevKitM-1） | ¥17.64 | 内置 TWAI + USB-Serial-JTAG |
| 收发器 | **SN65HVD230** | ¥10 | ⚠️ 必须 3.3V 版本 |
| 终端电阻 | 120Ω ×2 | ¥1 | 总线两端各一个 |
| 杜邦线 | 若干 | — | — |
| 合计 | | **约 ¥28** | |

### ⚠️ 收发器绝对是 3.3V 那颗

这是最容易翻车的一步。

| 型号 | 供电 | 能否配合 ESP32-C3 |
|:--|:--:|:--|
| **SN65HVD230** | 3.3V | ✅ 唯一正确选择 |
| TJA1050 | 5V | ❌ IO 电平不兼容，可能损坏 |
| MCP2551 | 5V | ❌ 同上 |
| TJA1051T/3 | 3.3V | ✅ 可以（带 /3 后缀的才是 3.3V） |

买的时候看丝印和商品标题，**"3.3V CAN 收发器" 是唯一判据**。5V 模块加上去，轻则通信全失败、TEC 快速涨到 255，重则烧 IO。

### 接线表

```
ESP32-C3                    SN65HVD230 模块
────────                    ───────────────
3V3       ────────────────► VCC   (3.3V)
GND       ────────────────► GND
GPIO4     ────────────────► CTX / TXD   (C3 发 → 收发器收)
GPIO5     ────────────────► CRX / RXD   (C3 收 ← 收发器发)
                            CANH ──┬──── 总线 CANH
                            CANL ──┴──── 总线 CANL
                            Rs   ─────── GND   (高速模式)
```

### 引脚分配的禁区

ESP32-C3 的 TWAI 通过 GPIO Matrix 路由，**几乎任意引脚都能映射**，但要避开这些：

| 引脚 | 用途 | 后果 |
|:--|:--|:--|
| `GPIO11` | VDD_SPI（Flash 供电） | 改了会掉 Flash |
| `GPIO12–17` | 封装内 Flash（SPI0/1） | 完全不可用 |
| `GPIO18/19` | USB-Serial-JTAG | 占用后无法烧录/调试 |
| `GPIO20/21` | UART0（默认日志口） | 占用后看不到 printf |
| `GPIO2/8/9` | strapping 脚（9=BOOT） | 上电电平影响启动模式 |

推荐组合：

| TX | RX | 备注 |
|:--:|:--:|:--|
| `GPIO4` | `GPIO5` | 最常用，多数板子这两个都引出了 |
| `GPIO6` | `GPIO7` | 备选 |
| `GPIO0` | `GPIO1` | 注意 0 是 strapping，谨慎 |

> 上板前先看自己板子的丝印 / 引脚图，确认 `GPIO4/5` 没被 RGB LED、按键之类占用。

### 三个致命坑

1. **终端电阻必须"两端各一个"**
   不是"一端两个"。总线物理两端各 120Ω，中间节点不加。两个 120Ω 并联才等于 60Ω 的标准特征阻抗。短线台架只加一个也能通，但长线或高波特率会随机丢帧。

2. **必须共地**
   ESP32-C3 的 GND 和 F407 / ZLG 的 GND 要连起来。不共地时 CANH/CANL 的差分电平整体漂移，表现是**间歇性错误帧，时好时坏**，极难排查。

3. **`Rs` 引脚接地 = 高速模式**
   SN65HVD230 的 `Rs`（斜率控制）接地是高速模式；接高电平或串电阻会降速。多数成品模块已把 Rs 接地或做了跳线，注意别接错。

---

## 三、开发环境

### 选型：ESP-IDF 而不是 Arduino

| 对比项 | ESP-IDF | Arduino |
|:--|:--|:--|
| TWAI 底层控制 | ✅ 直接调驱动，能看错误计数 | ⚠️ 封装掉细节 |
| 错误帧 / 状态机 | ✅ 完整暴露 | ❌ 隐藏 |
| 时序精度 | ✅ FreeRTOS 任务 + 精确延时 | ⚠️ `loop()` 不定时 |
| 上手速度 | 慢 | 快 |

**做对端节点必须能看见 TWAI 状态机**（错误计数、总线状态），否则注入故障时你不知道是"我发错了"还是"总线本来就有问题"。所以走 IDF。

### 安装

```bash
# Linux
mkdir -p ~/esp && cd ~/esp
git clone -b v5.3 --recursive https://github.com/espressif/esp-idf.git
cd esp-idf && ./install.sh esp32c3
. ./export.sh        # 每次开新终端都要 source
```

验证：

```bash
idf.py --version
# ESP-IDF v5.3.x
```

### 创建工程

```bash
cd ~/esp
idf.py create-project uds-peer
cd uds-peer
idf.py set-target esp32c3
```

### 关键 menuconfig 项

```bash
idf.py menuconfig
```

必须确认：

| 路径 | 设置 |
|:--|:--|
| `Component config → ESP-TWAI` | `Enable TWAI driver` = y |
| `Component config → FreeRTOS → Kernel` | Tick rate = 1000 Hz（提高时序精度） |
| `Serial flasher config → Flash size` | 按板子实际（常见 4MB） |

> Tick 改成 1000 Hz 是因为 UDS 的 `STmin` 最小是 1ms 量级，默认 100Hz（10ms/tick）精度不够。

### 烧录与监控

```bash
idf.py -p /dev/ttyUSB0 flash monitor
```

ESP32-C3 板子若用**内置 USB-Serial-JTAG**（多数 SuperMini），端口是 `/dev/ttyACM0`，且 IDF 会自动识别。

---

## 四、TWAI 驱动层

### 初始化

```c
#include "driver/twai.h"
#include "esp_log.h"

#define PIN_CAN_TX  4
#define PIN_CAN_RX  5

static const char *TAG = "can";

void can_init(void)
{
    twai_general_config_t g = TWAI_GENERAL_CONFIG_DEFAULT(
        PIN_CAN_TX, PIN_CAN_RX, TWAI_MODE_NORMAL);
    g.tx_queue_len = 16;
    g.rx_queue_len = 32;          // 高负载防丢帧
    g.alerts_enabled = TWAI_ALERT_ALL;
    g.clkout_divider = 0;

    twai_timing_config_t t = TWAI_TIMING_CONFIG_500KBITS();
    twai_filter_config_t f = TWAI_FILTER_CONFIG_ACCEPT_ALL();

    ESP_ERROR_CHECK(twai_driver_install(&g, &t, &f));
    ESP_ERROR_CHECK(twai_start());
    ESP_LOGI(TAG, "TWAI started @500kbps TX=%d RX=%d", PIN_CAN_TX, PIN_CAN_RX);
}
```

波特率宏，按需替换：

```c
TWAI_TIMING_CONFIG_125KBITS()
TWAI_TIMING_CONFIG_250KBITS()
TWAI_TIMING_CONFIG_500KBITS()   // UDS 最常用
TWAI_TIMING_CONFIG_1MBITS()     // 刷写常用
```

### 三种工作模式

这是 ESP32-C3 相对普通 MCU 的实用之处：

| 模式 | 宏 | 用途 |
|:--|:--|:--|
| 正常 | `TWAI_MODE_NORMAL` | 能发能收，需要总线上有节点回 ACK |
| **免应答** | `TWAI_MODE_NO_ACK` | **单板自测**，自发自收，不需要别人回 ACK |
| **只听** | `TWAI_MODE_LISTEN_ONLY` | 只收不发、不回 ACK，不干扰总线，当被动抓包器 |

**`NO_ACK` 是调试神器**：只有一块板 + 一个收发器、没有第二方时，它能让你验证「C3 + 收发器 + 接线」本身是否正常。硬件问题在这一步就隔离掉，后面排查就只剩软件。

**`LISTEN_ONLY` 也很实用**：把它接到车 / 台架上，它完全不参与通信（连 ACK 位都不发），只把报文转到 USB。比 ZLG 更轻量。

### 收发

```c
// 发送
bool can_send(uint32_t id, const uint8_t *data, uint8_t len)
{
    twai_message_t msg = {
        .identifier       = id,
        .data_length_code = len,
        .extd             = 0,      // 0 = 标准帧（11 位）
        .rtr              = 0,
    };
    memcpy(msg.data, data, len);
    return twai_transmit(&msg, pdMS_TO_TICKS(50)) == ESP_OK;
}

// 接收
bool can_recv(twai_message_t *out, uint32_t timeout_ms)
{
    return twai_receive(out, pdMS_TO_TICKS(timeout_ms)) == ESP_OK;
}
```

> ⚠️ `extd` 字段：**标准帧必须写 0**。写成 1 的话 ID 会按扩展帧（29 位）解析，对端根本收不到。这是最常见的"发出去但没反应"原因。

### 状态监控（很重要）

```c
void can_monitor_task(void *arg)
{
    while (1) {
        twai_status_info_t st;
        if (twai_get_status_info(&st) == ESP_OK) {
            if (st.state != TWAI_STATE_RUNNING) {
                ESP_LOGW(TAG, "TWAI state=%d", st.state);
            }
            if (st.tx_error_counter > 100 || st.rx_error_counter > 100) {
                ESP_LOGW(TAG, "ERR TEC=%lu REC=%lu bus_err=%lu",
                         st.tx_error_counter,
                         st.rx_error_counter,
                         st.bus_error_count);
            }
        }
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

判读方法：

| 现象 | 含义 | 排查方向 |
|:--|:--|:--|
| `TEC` 快速上涨到 255 | 发不出去，没人 ACK | 收发器接线 / 有无第二个节点 |
| `REC` 上涨 | 收到错误帧 | 波特率不一致 / 终端电阻 |
| `state = BUS_OFF` | 总线关闭（错误计数溢出） | 上述两者之一，需 `twai_initiate_recovery()` |
| `arbitration_lost_count` 高 | ID 冲突 | 两个节点 ID 重复 |

---

## 五、UDS 服务层

### 服务分发骨架

```c
#define ID_REQ  0x7E0
#define ID_RESP 0x7E8

void uds_task(void *arg)
{
    twai_message_t rx;
    while (1) {
        if (!can_recv(&rx, 100)) continue;
        if (rx.extd || rx.identifier != ID_REQ) continue;
        if (rx.data_length_code < 2) continue;

        uint8_t pci = rx.data[0] >> 4;
        if (pci != 0x0) {
            // 多帧，交给 ISO-TP 层（见第六节）
            isotp_on_frame(&rx);
            continue;
        }

        uint8_t sf_len = rx.data[0] & 0x0F;
        if (sf_len < 1) continue;

        uint8_t sid = rx.data[1];
        uds_dispatch(sid, &rx.data[2], sf_len - 1);
    }
}

void uds_dispatch(uint8_t sid, const uint8_t *param, uint8_t plen)
{
    switch (sid) {
    case 0x10: handle_session_control(param, plen); break;
    case 0x11: handle_ecu_reset(param, plen);       break;
    case 0x22: handle_read_did(param, plen);        break;
    case 0x27: handle_security_access(param, plen); break;
    case 0x2E: handle_write_did(param, plen);       break;
    case 0x31: handle_routine_control(param, plen); break;
    case 0x3E: handle_tester_present(param, plen);  break;
    default:
        send_negative(sid, NRC_SERVICE_NOT_SUPPORTED); // 0x11
        break;
    }
}
```

### 响应构造工具

```c
static void send_positive(uint8_t sid, const uint8_t *data, uint8_t len)
{
    uint8_t buf[8];
    uint8_t total = 1 + len;
    if (total > 7) {
        // 超 7 字节走多帧，见第六节
        isotp_send(ID_RESP, sid + 0x40, data, len);
        return;
    }
    buf[0] = total;               // SF 长度
    buf[1] = sid + 0x40;          // 肯定响应 SID = 请求 SID + 0x40
    memcpy(&buf[2], data, len);
    can_send(ID_RESP, buf, total + 1);
}

static void send_negative(uint8_t sid, uint8_t nrc)
{
    uint8_t buf[4] = { 0x03, 0x7F, sid, nrc };
    can_send(ID_RESP, buf, 4);
}
```

### NRC 码速查（对端必须能准确报出这些）

| NRC | 名称 | 触发场景 |
|:--:|:--|:--|
| `0x10` | generalReject | 通用拒绝 |
| `0x11` | serviceNotSupported | SID 不认识 |
| `0x12` | subFunctionNotSupported | 子功能不支持 |
| `0x13` | incorrectMessageLengthOrInvalidFormat | 长度不对 |
| `0x22` | conditionsNotCorrect | 状态不满足（如未进扩展会话就写） |
| `0x24` | requestSequenceError | 请求顺序错（没解锁就写） |
| `0x31` | requestOutOfRange | DID / 例程 ID 不存在 |
| `0x33` | securityAccessDenied | 未解锁 |
| `0x35` | invalidKey | 密钥错误 |
| `0x36` | exceededNumberOfAttempts | 尝试次数超限 |
| `0x37` | requiredTimeDelayNotExpired | 延时未到 |
| `0x71` | transferDataSuspended | 连续帧序号错 |
| `0x72` | generalProgrammingFailure | 编程失败（擦/写） |
| `0x78` | requestCorrectlyReceivedResponsePending | **处理中，稍后给正式响应** |

### 各服务的实现要点

#### 0x10 DiagnosticSessionControl

```c
static void handle_session_control(const uint8_t *p, uint8_t len)
{
    if (len < 1) { send_negative(0x10, 0x13); return; }
    uint8_t sub = p[0] & 0x7F;      // 去掉 suppressPosRsp 位

    switch (sub) {
    case 0x01: case 0x02: case 0x03:
        g_session = sub;
        g_s3_timer_reset();         // 关键：重置 S3 计时
        {
            uint8_t rsp[6] = {
                sub,
                0x00, 0x32,          // P2Server_max = 50ms（单位 1ms）
                0x01, 0xF4           // P2*Server_max = 5000ms（单位 10ms）
            };
            send_positive(0x10, rsp, 5);
        }
        break;
    default:
        send_negative(0x10, 0x12);
    }
}
```

> **P2 参数单位**：`P2Server_max` 以 **毫秒** 为单位（0x0032 = 50ms）；`P2*Server_max` 以 **10 毫秒** 为单位（0x01F4 = 5000ms）。这个单位差异是很多实现出错的地方。

#### 0x3E TesterPresent

```c
static void handle_tester_present(const uint8_t *p, uint8_t len)
{
    g_s3_timer_reset();
    uint8_t sub = len ? p[0] : 0;
    if (sub & 0x80) return;         // suppressPosRsp：不回
    uint8_t rsp[1] = { sub & 0x7F };
    send_positive(0x3E, rsp, 1);
}
```

#### 0x27 SecurityAccess

这是**最有价值的一个**，因为它逼你实现状态机：

```c
static void handle_security_access(const uint8_t *p, uint8_t len)
{
    if (len < 1) { send_negative(0x27, 0x13); return; }
    uint8_t sub = p[0];

    if ((sub & 1) == 1) {
        // 奇数次 = 请求种子
        if (g_unlocked) { send_negative(0x27, 0x22); return; }
        g_seed = esp_random();
        uint8_t rsp[4] = {
            sub,
            (g_seed >> 24) & 0xFF, (g_seed >> 16) & 0xFF,
            (g_seed >> 8)  & 0xFF
        };
        send_positive(0x27, rsp, 4);
    } else {
        // 偶数次 = 发送密钥
        if (len < 4) { send_negative(0x27, 0x13); return; }
        uint32_t key = ((uint32_t)p[1] << 24) | ((uint32_t)p[2] << 16)
                     | ((uint32_t)p[3] << 8)  |  (uint32_t)p[4];
        if (key == calc_key(g_seed)) {
            g_unlocked = true;
            uint8_t rsp[1] = { sub };
            send_positive(0x27, rsp, 1);
        } else {
            g_wrong_key_count++;
            if (g_wrong_key_count >= 3) {
                send_negative(0x27, 0x36);   // 次数超限
                g_lockout_until = xTaskGetTickCount() + pdMS_TO_TICKS(10000);
            } else {
                send_negative(0x27, 0x35);   // 密钥错
            }
        }
    }
}
```

**密钥算法故意做简单**（对端不是安全产品，目的是让被测端能算对）：

```c
static uint32_t calc_key(uint32_t seed)
{
    return seed ^ 0xA5A5A5A5;       // 换成任意确定性函数
}
```

> 真实 ECU 用 AES-128 或厂商私有算法。对端不需要真实，只需要**确定**——被测端能算出同一个值就行。

#### 0x22 ReadDataByIdentifier（需要多帧）

```c
static void handle_read_did(const uint8_t *p, uint8_t len)
{
    if (len < 2) { send_negative(0x22, 0x13); return; }
    uint16_t did = (p[0] << 8) | p[1];

    switch (did) {
    case 0xF190: {                  // VIN，17 字节 → 必须多帧
        uint8_t rsp[19];
        rsp[0] = 0xF1; rsp[1] = 0x90;
        memcpy(&rsp[2], "LSVAA1234567890AB", 17);
        send_positive(0x22, rsp, 19);   // 19 > 7 → 走 ISO-TP
        break;
    }
    case 0xF187: {                  // 零件号，短，单帧
        uint8_t rsp[6] = { 0xF1, 0x87, 'P', 'N', '0', '1' };
        send_positive(0x22, rsp, 6);
        break;
    }
    default:
        send_negative(0x22, 0x31);
    }
}
```

> `0xF190` 读 VIN 是**逼你实现 ISO-TP 的经典用例**，因为 VIN 固定 17 字节，响应必然超过单帧的 7 字节上限。

#### 0x78 responsePending（测超时的关键）

```c
static void handle_routine_control_slow(const uint8_t *p, uint8_t len)
{
    // 先回 0x78 说"在处理"
    send_negative(0x31, 0x78);

    // 然后慢慢干（这里用 3 秒模拟擦除 Flash）
    vTaskDelay(pdMS_TO_TICKS(3000));

    // 再给正式响应
    uint8_t rsp[4] = { 0x01, 0x02, 0x03, 0x00 };
    send_positive(0x31, rsp, 4);
}
```

**这是压测中最重要的场景**：被测端的 `P2*Server_max`（5000ms）如果没实现，这里就会误判为超时失败。

---

## 六、ISO-TP 多帧

### 四种 PCI 帧类型

| 类型 | 高 4 bit | 格式 | 长度字段 |
|:--|:--:|:--|:--|
| **SF** 单帧 | `0x0` | `0L DD DD...` | 低 4 bit = 数据长度（1–7） |
| **FF** 首帧 | `0x1` | `1L LL DD...` | 低 4 bit + 第二字节 = 总长度（12 bit，最大 4095） |
| **CF** 连续帧 | `0x2` | `2S DD DD...` | 低 4 bit = 序号（0–15 循环） |
| **FC** 流控帧 | `0x3` | `3F BS ST` | F = 流状态，BS = 块大小，ST = 最小间隔 |

### 流控帧细节

```
第三字节：3F BS STmin
               │  │   └── 最小间隔：0x00-0x7F = 毫秒；0xF1-0xF9 = 100-900 微秒
               │  └────── 块大小：允许连续发几帧后需再等 FC，0 = 不限
               └───────── 流状态：
                            0 = ContinueToSend (CTS) 继续发
                            1 = Wait (WT)            等我
                            2 = Overflow (OVFLW)     缓冲不够，别发了
```

### 发送侧（对端回长响应）

```c
typedef struct {
    uint8_t  buf[64];
    uint16_t total;
    uint16_t sent;
    uint8_t  next_seq;
    uint8_t  block_size;
    uint8_t  stmin_ms;
    bool     active;
} isotp_tx_t;

static isotp_tx_t g_tx;

void isotp_send(uint32_t id, uint8_t sid, const uint8_t *data, uint8_t len)
{
    uint8_t full[64];
    uint8_t flen = 0;
    full[flen++] = sid + 0x40;
    memcpy(&full[flen], data, len);
    flen += len;

    if (flen <= 7) {                       // 单帧
        uint8_t buf[8] = { flen };
        memcpy(&buf[1], full, flen);
        can_send(id, buf, flen + 1);
        return;
    }

    // 首帧：低 4bit 是总长高 4 位，第二字节是低 8 位
    g_tx.total = flen;
    g_tx.sent  = 6;
    g_tx.next_seq = 1;
    g_tx.active = true;

    uint8_t ff[8];
    ff[0] = 0x10 | ((flen >> 8) & 0x0F);
    ff[1] = flen & 0xFF;
    memcpy(&ff[2], full, 6);
    can_send(id, ff, 8);

    // 等对端 FC（见 isotp_rx_fc），拿到 BS/STmin 后继续
}

void isotp_continue_tx(uint32_t id)
{
    while (g_tx.active && g_tx.sent < g_tx.total) {
        uint8_t cf[8];
        uint8_t n = g_tx.total - g_tx.sent;
        if (n > 7) n = 7;

        cf[0] = 0x20 | (g_tx.next_seq & 0x0F);
        memcpy(&cf[1], &g_tx.buf[g_tx.sent], n);
        if (n < 7) memset(&cf[1 + n], 0xCC, 7 - n);   // 填充（可选）

        can_send(id, cf, 8);
        g_tx.sent += n;
        g_tx.next_seq++;

        if (g_tx.stmin_ms) vTaskDelay(pdMS_TO_TICKS(g_tx.stmin_ms));
    }
    g_tx.active = false;
}
```

### 接收侧（对端收被测端的长请求）

```c
typedef struct {
    uint8_t  buf[64];
    uint16_t total;
    uint16_t got;
    uint8_t  next_seq;
    bool     active;
} isotp_rx_t;

static isotp_rx_t g_rx;

void isotp_on_frame(const twai_message_t *f)
{
    uint8_t pci = f->data[0] >> 4;

    switch (pci) {
    case 0x0: {                                  // 单帧
        uint8_t n = f->data[0] & 0x0F;
        uds_dispatch(f->data[1], &f->data[2], n - 1);
        break;
    }
    case 0x1: {                                  // 首帧
        g_rx.total = ((f->data[0] & 0x0F) << 8) | f->data[1];
        memcpy(g_rx.buf, &f->data[2], 6);
        g_rx.got = 6;
        g_rx.next_seq = 1;
        g_rx.active = true;

        // 回 FC：CTS，块大小 0（不限），STmin 0
        uint8_t fc[3] = { 0x30, 0x00, 0x00 };
        can_send(ID_RESP, fc, 3);
        break;
    }
    case 0x2: {                                  // 连续帧
        if (!g_rx.active) { send_negative(0x00, 0x72); break; }
        uint8_t seq = f->data[0] & 0x0F;
        if (seq != (g_rx.next_seq & 0x0F)) {
            g_rx.active = false;
            send_negative(0x00, 0x71);           // transferDataSuspended
            break;
        }
        uint8_t n = g_rx.total - g_rx.got;
        if (n > 7) n = 7;
        memcpy(&g_rx.buf[g_rx.got], &f->data[1], n);
        g_rx.got += n;
        g_rx.next_seq++;

        if (g_rx.got >= g_rx.total) {
            g_rx.active = false;
            uds_dispatch(g_rx.buf[0], &g_rx.buf[1], g_rx.total - 1);
        }
        break;
    }
    case 0x3:                                    // 对端的流控（我们发长响应时收到）
        g_tx.block_size = f->data[1];
        g_tx.stmin_ms   = (f->data[2] <= 0x7F) ? f->data[2] : 1;
        if ((f->data[0] & 0x0F) == 0) isotp_continue_tx(ID_RESP);
        break;
    }
}
```

### 何时可以不上 ISO-TP

**第一批只做 SF 完全够用**：

| 服务 | 请求长度 | 响应长度 | 需要 MF？ |
|:--|:--:|:--:|:--:|
| `0x10` | 2 | 6 | ❌ |
| `0x3E` | 2 | 2 | ❌ |
| `0x11` | 2 | 2 | ❌ |
| `0x27`（种子） | 2 | 5 | ❌ |
| `0x22`（短 DID） | 3 | ≤7 | ❌ |
| `0x22`（VIN） | 3 | 19 | ✅ |
| `0x31` | ≤7 | ≤7 | ❌ |
| `0x36`（传固件） | — | — | ✅ |

**建议顺序**：先手写 SF 分支跑通 0x10 / 0x3E / 0x11 / 0x7F，等要测 VIN 或多帧刷写时再补 FF/CF/FC。手写 SF 的过程能让你看清 PCI 到底在干什么，比直接引库学到的多。

---

## 七、时序参数（三个必做项）

这三个参数不对，被测端的 UDS 栈 bug 永远测不出来——**因为大部分 bug 就在超时处理里**。

### P2Server_max

- **含义**：收到请求到"首个响应字节"的最长允许时间
- **对端要求**：正常响应必须在 50ms 内发出
- **被测端**：超过 50ms 没收到，应认为超时

```c
// 对端实现要注意：任何处理超过 50ms 的操作都必须先回 0x78
```

### P2*Server_max

- **含义**：发了 `0x78`（responsePending）之后，放宽到的最长等待时间
- **值**：5000ms = 5 秒
- **对端要求**：每次发 `0x78` 后重新计时

```c
// 长操作要周期性发 0x78，间隔小于 5000ms
void long_operation(void)
{
    for (int i = 0; i < 10; i++) {
        vTaskDelay(pdMS_TO_TICKS(3000));
        send_negative(0x31, 0x78);      // 每 3 秒续一次
    }
    send_positive(0x31, rsp, 4);
}
```

### S3Server

- **含义**：非默认会话下，多久没收到 `0x3E` 就自动回默认会话
- **值**：5000ms
- **对端要求**：必须实现，否则被测端"会话保活"逻辑测不出来

```c
static void s3_timer_task(void *arg)
{
    while (1) {
        vTaskDelay(pdMS_TO_TICKS(500));
        if (g_session != SESSION_DEFAULT) {
            if (xTaskGetTickCount() - g_last_activity > pdMS_TO_TICKS(5000)) {
                g_session = SESSION_DEFAULT;
                g_unlocked = false;
                ESP_LOGW(TAG, "S3 timeout -> back to default session");
            }
        }
    }
}
```

### 参数速查

| 参数 | 值 | 单位 | 谁来保证 |
|:--|:--:|:--|:--|
| `P2Server_max` | 50 | ms | 响应方（对端） |
| `P2*Server_max` | 5000 | ms | 响应方（对端） |
| `S3Server` | 5000 | ms | 响应方（对端） |
| `STmin` | 0–127 | ms | 接收方（发 FC 时指定） |
| `N_Bs`（等 FC） | 1000 | ms | 发送方超时 |
| `N_Cr`（等 CF） | 1000 | ms | 接收方超时 |

---

## 八、联调四步走

### 第 1 步：单板自测（不需要任何外部设备）

```c
// 换成 NO_ACK 模式
twai_general_config_t g = TWAI_GENERAL_CONFIG_DEFAULT(
    PIN_CAN_TX, PIN_CAN_RX, TWAI_MODE_NO_ACK);
```

发一帧、收一帧，能收回来就说明：

- ✅ ESP32-C3 的 TWAI 外设正常
- ✅ 收发器的 TX/RX 接线正确
- ✅ 引脚映射没问题

**这一步把硬件问题全部隔离掉**。如果这里就失败，别往下走，先查接线。

判据：
```c
// TEC/REC 保持 0
twai_status_info_t st;
twai_get_status_info(&st);
assert(st.tx_error_counter == 0);
```

### 第 2 步：接 ZLG 抓包（验证物理层）

C3 和 ZLG 盒子并联到总线，ZLG 侧：

```python
# 用 python-can（zlgcan backend）
import can
bus = can.Bus(interface='zlgcan', channel=0, bitrate=500000)

# C3 周期性发 0x7E8，这里应该能收到
for msg in bus:
    print(hex(msg.arbitration_id), msg.data.hex())
```

能收到 → 物理层 + 仲裁 + ACK 全通。

> 注意：ZLG 会替你回 ACK 位。如果这步通了但第 1 步没过，说明是收发器方向问题。

### 第 3 步：与 F407 真实握手

F407 发 `0x7E0: 02 3E 00`，C3 回 `0x7E8: 02 7E 00`。

通了之后，你就有了**可反复重放的对端**，不必再靠日志猜对端行为。

验证序列（建议按此顺序）：

| 步骤 | 请求 | 期望响应 | 验证点 |
|:--:|:--|:--|:--|
| 1 | `02 3E 00` | `02 7E 00` | 基本链路 |
| 2 | `02 10 03` | `06 50 03 00 32 01 F4` | 会话切换 + 参数 |
| 3 | `03 22 F1 90` | 多帧 VIN | ISO-TP 收包 |
| 4 | `02 27 01` | `05 67 01 XX XX XX` | 种子 |
| 5 | `02 27 02 ...` | `02 67 02` 或 `03 7F 27 35` | 密钥校验 |
| 6 | `04 31 01 02 03` | `01 78` → 后 `04 71 01 02 03 00` | 0x78 延时 |

### 第 4 步：故障注入压测

见下一节。

---

## 九、故障注入压测（自建对端的核心价值）

**ZLG 只能被动记录，C3 能主动作恶。**这些场景你拿真 ECU 根本试不出来。

### 场景清单

| # | 注入内容 | 被测端应有行为 | 对端改动 |
|:--:|:--|:--|:--|
| 1 | 完全不响应 | P2Server 超时触发 | 收到请求后直接 return |
| 2 | `0x78` 后延迟 6 秒 | P2* 超时触发（>5000ms） | `delay(6000)` 再响应 |
| 3 | `0x78` 后延迟 3 秒 | 正常继续等 | `delay(3000)` |
| 4 | CF 序号跳号（1→3） | 丢弃整包 + NRC `0x71` | 发送时 `next_seq += 2` |
| 5 | 回 NRC `0x13` 长度错 | 记录并上报 | 故意回错 NRC |
| 6 | 回超长响应（撑爆缓冲） | 不崩溃、报错 | 发 100 字节响应 |
| 7 | 响应 SID 错（回 `0x51` 而非 `0x50`） | 判定为非法响应 | 改响应字节 |
| 8 | 流控帧回 `Overflow`（`0x32`） | 停止发送 | FC 第三字节 0x02 |
| 9 | 流控帧 `WAIT`（`0x31`） | 等待而非继续 | FC 第三字节 0x01 |
| 10 | STmin 设 127ms | 按间隔发送 | FC 第三字节 0x7F |
| 11 | 总线 `BUS_OFF` | 恢复机制 | 短接 CANH/CANL 制造 |
| 12 | 非默认会话下不发 `0x3E` | S3 超时回默认 | 对端 5 秒后自动回默认 |

### 实现示例：可配置的故障注入

```c
typedef struct {
    uint8_t  drop_next_n;        // 接下来 N 个请求不响应
    int32_t  delay_ms;           // 响应前延迟
    uint8_t  force_nrc;          // 非 0 则强制回此 NRC
    bool     corrupt_seq;        // CF 序号跳号
    uint8_t  fc_flow_status;     // 0=CTS 1=WT 2=OVFLW
} fault_config_t;

static fault_config_t g_fault;

void uds_task(void *arg)
{
    twai_message_t rx;
    while (1) {
        if (!can_recv(&rx, 100)) continue;
        if (rx.identifier != ID_REQ) continue;

        if (g_fault.drop_next_n > 0) {   // 注入 1：不响应
            g_fault.drop_next_n--;
            continue;
        }
        if (g_fault.delay_ms > 0) {      // 注入 2/3：延迟
            vTaskDelay(pdMS_TO_TICKS(g_fault.delay_ms));
        }
        if (g_fault.force_nrc) {         // 注入 5：强制 NRC
            send_negative(rx.data[1], g_fault.force_nrc);
            continue;
        }
        // 正常处理
        isotp_on_frame(&rx);
    }
}
```

**用串口命令动态改**：

```c
// 串口输入 "nrc 13" → 接下来所有响应都回 NRC 0x13
// 串口输入 "drop 3" → 接下来 3 个请求不响应
// 串口输入 "delay 6000" → 响应前延迟 6 秒
```

这样压测时不用反复烧固件。

---

## 十、坑清单

| # | 坑 | 表现 | 解法 |
|:--:|:--|:--|:--|
| 1 | 用 5V 收发器（TJA1050/MCP2551） | 完全不通或芯片发热 | 换 SN65HVD230 |
| 2 | 波特率不一致 | 错误帧刷屏，TEC 飙到 255 | 双方确认同一波特率 |
| 3 | 只有一端有 120Ω | 短线能通，长线随机丢帧 | 两端各一个 |
| 4 | 没共地 | 间歇性错误帧，时好时坏 | GND 必须连 |
| 5 | `extd` 标志没清 | 标准帧发成扩展帧，对端收不到 | 标准帧 `extd = 0` |
| 6 | 环形队列太小 | 高负载丢帧，`twai_receive` 超时 | `rx_queue_len` ≥ 32 |
| 7 | `twai_start` 在 `install` 前 | 断言失败重启 | 顺序：install → start |
| 8 | `P2` 和 `P2*` 单位搞混 | 对端报 50 / 5 而非 50ms / 5000ms | P2 用 ms，P2* 用 10ms |
| 9 | GPIO 选了 strapping 脚 | 上电偶发不进 boot | 避开 GPIO2/8/9 |
| 10 | 没实现 S3 超时 | 会话永远不退出 | 加 5 秒计时任务 |
| 11 | CF 序号从 1 开始但封装成 `0x21` | 对端解析失败 | `0x20 \| (seq & 0x0F)`，seq 从 1 起 |
| 12 | 首帧长度字段只写低 8 位 | >255 字节的包长度错 | `0x10 \| ((len>>8) & 0x0F)` 且第二字节 `len & 0xFF` |
| 13 | ISR 里调 `twai_transmit` | 阻塞或崩溃 | 用队列转到任务里发 |
| 14 | 任务栈太小 | 偶发 panic / Guru Meditation | TWAI 任务栈 ≥ 4096 |

---

## 十一、工程骨架

```
uds-peer/
├── CMakeLists.txt
├── sdkconfig
├── sdkconfig.defaults
└── main/
    ├── CMakeLists.txt
    ├── uds_peer.c          # app_main + 任务创建
    ├── can_drv.c/.h        # TWAI 初始化/收发/监控
    ├── uds_core.c/.h       # 服务分发 + NRC
    ├── uds_services.c/.h   # 各 SID 处理
    ├── isotp.c/.h          # 多帧收发
    ├── fault_inject.c/.h   # 故障注入
    └── cli.c/.h            # 串口命令
```

### 顶层 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(uds-peer)
```

### main/CMakeLists.txt

```cmake
idf_component_register(
    SRCS "uds_peer.c" "can_drv.c" "uds_core.c"
         "uds_services.c" "isotp.c" "fault_inject.c" "cli.c"
    INCLUDE_DIRS "."
    REQUIRES driver esp_driver_twai esp_timer
)
```

### sdkconfig.defaults

```ini
CONFIG_IDF_TARGET="esp32c3"
CONFIG_FREERTOS_HZ=1000
CONFIG_ESP_MAIN_TASK_STACK_SIZE=8192
CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y
```

### 任务划分

```c
void app_main(void)
{
    can_init();
    cli_init();

    xTaskCreate(can_rx_task,   "can_rx",   4096, NULL, 10, NULL);
    xTaskCreate(can_mon_task,  "can_mon",  3072, NULL,  3, NULL);
    xTaskCreate(s3_timer_task, "s3_timer", 2048, NULL,  2, NULL);

    ESP_LOGI(TAG, "UDS peer ready: REQ=0x%03X RESP=0x%03X", ID_REQ, ID_RESP);
}
```

### 内存占用估算

| 项 | 大小 |
|:--|:--:|
| 代码（含 IDF） | ~600 KB |
| ISOTP 缓冲（2×64B） | 128 B |
| 任务栈（3 个） | ~9 KB |
| TWAI 队列（16+32 帧） | ~4 KB |
| **合计** | **~650 KB / 4 MB Flash** |

余量充足，可以放心加日志和 CLI。

---

## 十二、一页速查

### 硬件

```
C3            收发器(SN65HVD230)
3V3 ───────── VCC
GND ───────── GND
GPIO4 ─────── CTX
GPIO5 ─────── CRX
              CANH ─┬─ 总线
              CANL ─┴─ 总线
              Rs ──── GND
```

### 标准 ID

| 用途 | ID |
|:--|:--|
| 物理请求 | `0x7E0` |
| 物理响应 | `0x7E8` |
| 功能请求 | `0x7DF` |

### PCI

| 类型 | 首字节 | 说明 |
|:--|:--|:--|
| SF | `0L` | L = 长度 1–7 |
| FF | `1H LL` | 总长 = (H<<8) \| LL |
| CF | `2S` | S = 序号 0–15 循环 |
| FC | `3F BS ST` | F=流状态 BS=块大小 ST=间隔 |

### 关键参数

| 参数 | 值 |
|:--|:--|
| `P2Server_max` | 50 ms |
| `P2*Server_max` | 5000 ms |
| `S3Server` | 5000 ms |
| `STmin` | 0–127 ms |

### 必备 NRC

`0x11` 服务不支持 · `0x12` 子功能不支持 · `0x13` 长度错 · `0x22` 条件不满足 · `0x31` 超范围 · `0x33` 未解锁 · `0x35` 密钥错 · `0x71` 传输暂停 · `0x78` 处理中

### 联调顺序

1. `NO_ACK` 单板自测 → 验硬件
2. 接 ZLG 抓包 → 验物理层
3. 与 F407 握手 `3E` / `10` → 验协议
4. 故障注入 12 场景 → 验健壮性

---

## 十三、参考

| 用途 | 资源 |
|:--|:--|
| 芯片手册 | ESP32-C3 Technical Reference Manual（第 28 章 TWAI） |
| 驱动 API | ESP-IDF `driver/twai.h` |
| 诊断协议 | ISO 14229-1（UDS）· ISO 15765-2（ISO-TP）· ISO 15765-4（CAN 诊断） |
| 配套文档 | 本库《ZLG CAN 库与 UDS 刷写详解》 |
| CAN 基础 | 本库《ZLG CAN 库与 UDS 刷写详解》第二至四章 |