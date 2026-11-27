---
{"dg-publish":true,"permalink":"/knowledge/embedded/04-ZLG-CAN库与UDS刷写详解/","title":"ZLG CAN 库与 UDS 刷写详解","tags":["CAN","CANFD","ZLG","周立功","USB-CAN","ControlCAN","zlgcan","接口库","二次开发","UDS","ISO-TP","ISO15765","刷写","Bootloader","嵌入式","兼容卡","创芯科技","canalystii","python-can"],"dg-note-properties":{"title":"ZLG CAN 库与 UDS 刷写详解","date":"2026-11-24","tags":["CAN","CANFD","ZLG","周立功","USB-CAN","ControlCAN","zlgcan","接口库","二次开发","UDS","ISO-TP","ISO15765","刷写","Bootloader","嵌入式","兼容卡","创芯科技","canalystii","python-can"]}}
---


# 🔧 ZLG CAN 库与 UDS 刷写详解

> 周立功的 USB-CAN 盒子在国内工控 / 汽车电子调试里占有率很高，但它的二次开发库有**两代**：
> 老的 **ControlCAN（`VCI_*` 接口，只支持 CAN 2.0）** 和新的 **zlgcan（`ZCAN_*` 接口，支持 CAN FD）**。
> 很多人踩坑就踩在「拿老库去驱动 FD 设备」或者「索引/通道号传错」上。本文把两代库的核心接口、结构体、调用顺序和坑点一次讲清。
> 最后还给出**基于 ZLG 盒子做 UDS 刷写（ISO 15765-2 + ISO 14229）的核心实现代码**，可直接改造成产线刷写工具。

---

## 📑 本页导航

| 章节 | 内容 |
|:--|:--|
| 一、先分清两代库 | `VCI_*` vs `ZCAN_*` 该用哪个 |
| 二、ControlCAN 核心接口 | 老库函数 / 结构体 / 波特率表 |
| 三、zlgcan 核心接口 | 新库函数 / 帧结构 / FD 标志位 |
| 四、非 ZLG 兼容卡 | 创芯科技 USBCAN-2A 这类卡能不能用、怎么接 |
| 五、经典 CAN 与 CAN FD | 你的 ECU 决定用哪个，FD 代码怎么降级 |
| 六、标准调用流程 | 打开 → 初始化 → 启动 → 收发 → 关闭 |
| 七、Python 侧接入 | `python-can` 的 zlgcan / canalystii backend |
| 八、常见坑 | 十条实战经验 |
| 九、UDS 刷写实现 | ISO-TP 分包 + 刷写时序 + 核心代码 |
| 十、一页速查 | 表格总结 |

---

## 一、先分清两代库

| 对比项 | ControlCAN（老） | zlgcan（新） |
|:--|:--|:--|
| 头文件 | `ControlCAN.h` | `zlgcan.h` |
| 函数前缀 | `VCI_` | `ZCAN_` |
| 设备句柄 | 无句柄，靠 `(DeviceType, DeviceInd, CANInd)` 三元组定位 | 返回 `DEVICE_HANDLE` / `CHANNEL_HANDLE` |
| CAN 2.0 | ✅ | ✅ |
| CAN FD | ❌ | ✅ |
| 波特率设置 | `Timing0/Timing1` 查表 | 直接填 `bitrate` / `dbitrate`（bps） |
| 多设备/多通道 | 索引易混 | 句柄化，清晰 |
| 动态库 | `ControlCAN.dll` / `libcontrolcan.so` | `zlgcan.dll` / `libzlgcan.so` |

**结论**：只要是**ZLG 原厂设备**，又不是维护十几年前的老代码，**一律用 zlgcan**。USBCAN-II / USBCANFD-200U 等新设备，官方给的都是 zlgcan 示例。

> ⚠️ 但如果你手上是**非 ZLG 的兼容卡**（创芯科技 USBCAN-2A、吉阳等），这条结论要反过来——它们只兼容老库的 `VCI_*` 接口，`libzlgcan` 根本认不出这种设备。详见第四章。

设备类型宏（`ZCANDeviceType`）常用的几个：

| 设备 | 宏 | 值 |
|:--|:--|--:|
| USBCAN-I | `ZCAN_USBCAN1` | 3 |
| USBCAN-II | `ZCAN_USBCAN2` | 4 |
| USBCAN-E-U | `ZCAN_USBCAN_E_U` | 20 |
| USBCAN-2E-U | `ZCAN_USBCAN_2E_U` | 21 |
| USBCAN-4E-U | `ZCAN_USBCAN_4E_U` | 31 |
| USBCANFD-100U | `ZCAN_USBCANFD_100U` | 42 |
| USBCANFD-200U | `ZCAN_USBCANFD_200U` | 41 |
| USBCANFD-400U | `ZCAN_USBCANFD_400U` | 76 |
| USBCANFD-800U | `ZCAN_USBCANFD_800U` | 59 |

> 完整表有 80+ 项（含 PCIe / 以太网 / WiFi / 离线设备），以实际 SDK 的 `zlgcan.h` 为准。

---

## 二、ControlCAN 核心接口（老库）

### 2.1 函数族

```c
unsigned long VCI_OpenDevice (unsigned long DeviceType, unsigned long DeviceInd, unsigned long Reserved);
unsigned long VCI_CloseDevice(unsigned long DeviceType, unsigned long DeviceInd);
unsigned long VCI_InitCAN    (unsigned long DeviceType, unsigned long DeviceInd,
                              unsigned long CANInd, PVCI_INIT_CONFIG pInitConfig);
unsigned long VCI_StartCAN   (unsigned long DeviceType, unsigned long DeviceInd, unsigned long CANInd);
unsigned long VCI_ResetCAN   (unsigned long DeviceType, unsigned long DeviceInd, unsigned long CANInd);
unsigned long VCI_ClearBuffer(unsigned long DeviceType, unsigned long DeviceInd, unsigned long CANInd);

unsigned long VCI_Transmit   (unsigned long DeviceType, unsigned long DeviceInd, unsigned long CANInd,
                              PVCI_CAN_OBJ pSend, unsigned long Len);
unsigned long VCI_Receive    (unsigned long DeviceType, unsigned long DeviceInd, unsigned long CANInd,
                              PVCI_CAN_OBJ pReceive, unsigned long Len, int WaitTime);

unsigned long VCI_GetReceiveNum(unsigned long DeviceType, unsigned long DeviceInd, unsigned long CANInd);
unsigned long VCI_ReadBoardInfo (unsigned long DeviceType, unsigned long DeviceInd, PVCI_BOARD_INFO pInfo);
unsigned long VCI_ReadErrInfo   (unsigned long DeviceType, unsigned long DeviceInd,
                                 unsigned long CANInd, PVCI_ERR_INFO pErrInfo);
unsigned long VCI_SetReference  (unsigned long DeviceType, unsigned long DeviceInd,
                                 unsigned long CANInd, unsigned long RefType, void *pData);
```

返回值统一：**`STATUS_OK = 1` 成功，`STATUS_ERR = 0` 失败**（不是 0 表示成功，这点和 POSIX 相反，极易写错）。

### 2.2 `VCI_CAN_OBJ`：收发帧结构

```c
typedef struct _VCI_CAN_OBJ {
    unsigned int  ID;          // 帧 ID，标准帧填 11 位，扩展帧填 29 位
    unsigned int  TimeStamp;   // 时间戳，单位 0.1 ms（设备上电起算）
    unsigned char TimeFlag;    // 时间戳是否有效：1 有效
    unsigned char SendType;    // 0 正常发送 / 1 单次发送 / 2 自发自收 / 3 单次自发自收
    unsigned char RemoteFlag;  // 0 数据帧 / 1 远程帧
    unsigned char ExternFlag;  // 0 标准帧 / 1 扩展帧
    unsigned char DataLen;     // 数据长度 0~8
    unsigned char Data[8];     // 数据域
    unsigned char Reserved[3];
} VCI_CAN_OBJ, *PVCI_CAN_OBJ;
```

要点：

- `ID` **靠右对齐**（不要左移），`ExternFlag` 决定按 11 位还是 29 位解析。
- `SendType` 只在发送时有效；`TimeFlag/TimeStamp` 只在接收时由硬件填。
- `DataLen` 是实际字节数，`Data[]` 只取前 `DataLen` 个有效。

### 2.3 `VCI_INIT_CONFIG`：初始化配置

```c
typedef struct _VCI_INIT_CONFIG {
    unsigned long AccCode;   // 验收码
    unsigned long AccMask;   // 屏蔽码，全 0xFFFFFFFF = 全收
    unsigned long Reserved;
    unsigned char Filter;    // 0 双滤波 / 1 单滤波
    unsigned char Timing0;   // 波特率定时器 0
    unsigned char Timing1;   // 波特率定时器 1
    unsigned char Mode;      // 0 正常 / 1 只听(静默) / 2 自发自收
} VCI_INIT_CONFIG, *PVCI_INIT_CONFIG;
```

### 2.4 波特率查表（SJA1000 定时器）

| 波特率 | Timing0 | Timing1 |
|:--|:--|:--|
| 10 kbps | 0x31 | 0x1C |
| 20 kbps | 0x18 | 0x1C |
| 50 kbps | 0x09 | 0x1C |
| 100 kbps | 0x04 | 0x1C |
| 125 kbps | 0x03 | 0x1C |
| 250 kbps | 0x01 | 0x1C |
| 500 kbps | 0x00 | 0x1C |
| 800 kbps | 0x00 | 0x16 |
| 1000 kbps | 0x00 | 0x14 |

### 2.5 板卡信息

```c
typedef struct _VCI_BOARD_INFO {
    unsigned short hw_Version, fw_Version, dr_Version, in_Version;
    unsigned short irq_Num;
    unsigned char  can_Num;          // 通道数
    char           str_Serial_Num[20];
    char           str_hw_Type[40];
    unsigned short Reserved[4];
} VCI_BOARD_INFO;
```

`VCI_ReadBoardInfo` 是最靠谱的"设备到底在不在"的验证手段，还可以用它读序列号给多设备排序。

---

## 三、zlgcan 核心接口（新库，推荐）

### 3.1 函数族与句柄模型

```c
// —— 设备级 ——
DEVICE_HANDLE  ZCAN_OpenDevice (UINT device_type, UINT device_index, UINT reserved);
UINT           ZCAN_CloseDevice(DEVICE_HANDLE device_handle);
UINT           ZCAN_GetDeviceInf(DEVICE_HANDLE device_handle, ZCAN_DEV_INF *pInfo);

// —— 通道级 ——
CHANNEL_HANDLE ZCAN_InitCAN (DEVICE_HANDLE device_handle, UINT can_index, ZCAN_INIT *pInit);
UINT           ZCAN_StartCAN(CHANNEL_HANDLE channel_handle);
UINT           ZCAN_ResetCAN(CHANNEL_HANDLE channel_handle);
UINT           ZCAN_ClearBuffer(CHANNEL_HANDLE channel_handle);

// —— 收发 ——
UINT ZCAN_Transmit    (CHANNEL_HANDLE h, ZCAN_Transmit_Data *pTransmit, UINT len);
UINT ZCAN_TransmitFD  (CHANNEL_HANDLE h, ZCAN_TransmitFD_Data *pTransmit, UINT len);
UINT ZCAN_Receive     (CHANNEL_HANDLE h, ZCAN_Receive_Data *pReceive, UINT len, INT wait_time);
UINT ZCAN_ReceiveFD   (CHANNEL_HANDLE h, ZCAN_ReceiveFD_Data *pReceive, UINT len, INT wait_time);
UINT ZCAN_GetReceiveNum(CHANNEL_HANDLE h, UINT type);

// —— 通用属性（终端电阻 / 波特率 / ISO 标准 / 滤波）——
UINT ZCAN_SetValue(DEVICE_HANDLE h, CHAR *path, void *value);
UINT ZCAN_GetValue(DEVICE_HANDLE h, CHAR *path, void *value);
```

核心变化：**一切靠句柄**。`ZCAN_OpenDevice` 拿设备句柄，`ZCAN_InitCAN` 拿通道句柄，之后收发只用通道句柄，不用再传三元组，多设备多通道不会串。

注意：`ZCAN_*` 系列成功返回 **1**、失败返回 **0**，与老库保持一致（不是句柄非空即成功——句柄为 0 也是失败）。

### 3.2 `ZCAN_INIT`：通道初始化

```c
typedef struct {
    UINT  acc_code;    // 验收码
    UINT  acc_mask;    // 屏蔽码，0xFFFFFFFF = 全收
    UINT  reserved;
    UINT  filter;      // 0 双滤波 / 1 单滤波
    UINT  timing0;     // CAN 2.0 时用（同 SJA1000 定时器）
    UINT  timing1;
    UINT  mode;        // 0 正常 / 1 只听
} ZCAN_INIT;
```

对 **CAN FD 设备**，更常用的是走 `ZCAN_SetValue` 设波特率，或者直接调用 SDK 提供的 `ZCAN_InitCAN` 前先设好 `canfd_abit_baud_rate` / `canfd_dbit_baud_rate`。

### 3.3 帧结构与 CAN FD 标志位

```c
typedef struct {
    UINT  can_id;     // ID（含扩展帧标志位）
    UINT  can_dlc;    // 数据长度：CAN 0~8，CANFD 0~64
    UINT  __pad;
    UINT  __res0;
    UINT  __res1;
    BYTE  data[64];   // 统一 64 字节，CAN 只取前 8
} zcan_frame;
```

发送时用带 `flags` 的版本：

```c
#define CANFD_FDF  0x01   // FD 格式帧
#define CANFD_BRS  0x02   // 数据段波特率切换
#define CANFD_ESI  0x04   // 错误状态指示（被动错误）
```

```c
typedef struct {
    UINT       transmit_type;  // 0 正常 / 1 单次 / 2 自发自收 / 3 单次自发自收
    UINT       frame_reserved;
    zcan_frame frame;
} ZCAN_Transmit_Data;

typedef struct {
    UINT       transmit_type;
    UINT       frame_reserved;
    zcan_frame frame;          // 且 frame 需带 flags = CANFD_FDF | CANFD_BRS
} ZCAN_TransmitFD_Data;
```

**发 CAN FD 的必做三件事**：① 用 `ZCAN_TransmitFD`；② `flags` 置 `CANFD_FDF`（要变速再加 `CANFD_BRS`）；③ `can_dlc` 填 0~64 的实际长度。

### 3.4 `ZCAN_SetValue` 常用路径

| 用途 | path | value |
|:--|:--|:--|
| 打开通道 0 终端电阻 | `"0"` | `"1"` |
| 关闭通道 0 终端电阻 | `"0"` | `"0"` |
| 打开通道 1 终端电阻 | `"1"` | `"1"` |
| FD 仲裁段波特率 | `"canfd_abit_baud_rate"` | `"500000"` |
| FD 数据段波特率 | `"canfd_dbit_baud_rate"` | `"2000000"` |
| FD 标准选择 | `"canfd_standard"` | `"0"`=ISO / `"1"`=非 ISO |

> 终端电阻的 path 就是**通道号字符串**，是最容易漏的一步：默认常常是关的，双机对连收不到帧先查这里。
> 具体 path 名称随设备型号 / SDK 版本略有差异，以官方 `zlgcan` 文档的「设备属性表」为准。

---

## 四、非 ZLG 兼容卡能不能用

市面上大量国产 USB-CAN 盒子写着「兼容 ZLG」，价格只有原厂的 1/3。典型代表是**创芯科技（Chuangxin）的 USBCAN-2A / CANalyst-II**、吉阳等。

关键是要先分清「兼容」到底兼容哪一层：

| 兼容层次 | 含义 | 实际影响 |
|:--|:--|:--|
| **API 兼容** | 厂商提供同名的 `ControlCAN.h` + `libcontrolcan.so` / `ControlCAN.dll`，函数签名和 `VCI_CAN_OBJ` 结构体一致 | 源码不用改，重新编译即可 |
| **设备兼容** | ZLG 的 `libzlgcan.so` 能直接认这块卡 | **基本不存在**，`ZCAN_OpenDevice` 返回 0 |

所以「兼容 ZLG」几乎总是指**第一层**：它模仿的是老库 `VCI_*` 的接口，不是新库 `ZCAN_*`，更不是 ZLG 的 USB 协议。

### 4.1 一张判定表：本文各章对兼容卡是否适用

以 USBCAN-2A（CAN 2.0，双通道）为例：

| 章节 | 能否直接用 | 原因 |
|:--|:--:|:--|
| 一、两代库之分 | ⚠️ 反着读 | 兼容卡只能用**老库** |
| 二、ControlCAN 接口 | ✅ | 正是兼容卡的接口 |
| 三、zlgcan 接口 | ❌ | `libzlgcan` 认不出非 ZLG 设备 |
| 五、经典 CAN 与 CAN FD | ✅ | 兼容卡全是 CAN 2.0 |
| 六、标准调用流程 | ✅ | 用 ControlCAN 那半边骨架 |
| 七、Python 侧接入 | ⚠️ 换后端 | 不能用 `zlgcan` backend，见 4.4 |
| 八、常见坑 | ✅ | 十条全适用 |
| 九、UDS 刷写 | ⚠️ 改代码 | 协议层通用，但给的代码是 FD 版，见第五章 |
| 十、一页速查 | ✅ | 只看 ControlCAN 那一列 |

一句话：**能用，但只覆盖本文的一半，而且刷写代码要降级到经典 CAN。**

### 4.2 路线 A：继续用 ControlCAN（C 代码复用）

如果你已经有一份基于 `VCI_*` 的代码，换卡后基本零改动，只需三件事：

**① 确认 `DeviceType` 的值**

ZLG 原厂的宏是 `VCI_USBCAN1 = 3`、`VCI_USBCAN2 = 4`。兼容卡厂商可能改成自己的值（常见做法是照抄 3 / 4，但不保证）。

这个值**不要猜**，看随卡提供的 demo 源码里 `VCI_OpenDevice` 的第一个参数，或厂商 PDF 手册的设备类型表。

**② 波特率用 2.4 节的 Timings 表**

兼容卡内部同样是 SJA1000 风格的 BTR0/BTR1 寄存器，所以本文 2.4 节的表和它通用：

| 波特率 | Timing0 | Timing1 |
|:--|:--|:--|
| 250 kbps | `0x01` | `0x1C` |
| 500 kbps | `0x00` | `0x1C` |

**③ 先确认厂商给不给 Linux 库**

这是最大的现实风险：很多国产兼容卡**只提供 Windows 的 DLL**，Linux 的 `.so` 要么没有，要么只在某些型号上提供。收货后先打开安装包看一眼有没有 `libcontrolcan.so`，没有就只能走 4.3。

### 4.3 路线 B：python-can 的 canalystii 后端（推荐）

`python-can` 有一个**专门给创芯科技设备**的后端，不依赖厂商 DLL，是纯 Python + `pyusb` 实现（早期靠逆向，现在由 python-can 官方维护）：

```bash
pip install "python-can[canalystii]"
```

Windows 上还需要用 [Zadig](https://zadig.akeo.ie/) 把设备的 USB 驱动换成 `libusb-win32`；Linux 上装好 `libusb` 即可，但要注意 udev 权限。

用法和普通 `can.Bus` 一样：

```python
import can

bus = can.Bus(interface="canalystii", channel=0, bitrate=500000)

# 发一帧 UDS 请求：10 03（进入扩展会话）
bus.send(can.Message(arbitration_id=0x18DA10F1, is_extended_id=True,
                     data=[0x02, 0x10, 0x03]))

msg = bus.recv(timeout=1.0)
print(hex(msg.arbitration_id), msg.data.hex())

bus.shutdown()
```

这条路线的好处很实际：

- **跨平台一致**：Windows / Linux / macOS 都能跑，不用管厂商给不给 `.so`
- **不用处理 32/64 位库不匹配**（纯 Python，没有二进制依赖）
- **和本文第九章的 UDS 逻辑天然对接**，因为 `python-can` 的 `Message` 已经把扩展帧、远程帧抽象好了

### 4.4 canalystii 后端的已知限制

以下几条来自 python-can 官方文档和 `canalystii` 包源码，用之前必须知道：

| 限制 | 说明 | 应对 |
|:--|:--|:--|
| **不支持 CAN FD** | 后端强制 `CAN_20`，传 `BitTimingFd` 会被拒 | 兼容卡本来也只有 2.0，无影响 |
| **双通道时间戳可能乱序** | 通道 0 和 1 的帧在软件层返回顺序不确定（各自通道内部有序） | 需要跨通道比对时**按 timestamp 排序** |
| **时间戳单位是 100 µs** | 硬件给的是 0.1 ms 计数 | 拿来算时延前先乘 `100e-6` |
| **只认特定 VID/PID** | `0x04d8:0x0053`（Microchip 的 VID，卡上用的是 PIC32） | 设备管理器里核对，不符则后端找不到设备 |
| **波特率只支持表内值** | 5k / 10k / 20k / 50k / 100k / 125k / 250k / 500k / 800k / 1M 等 | 非标准波特率要传 `timing0/timing1` 原始值 |

> 若你的卡 VID/PID 不是 `04d8:0053`（有些兼容卡换了芯片），canalystii 后端就认不出来，只能回到 4.2 的 ControlCAN 路线。

**一个小技巧**：不确定卡是哪种协议时，先用 `lsusb` 看一眼 VID/PID，再决定走哪条路，比装完驱动再试要快。

### 4.5 收货后的验收清单

1. `lsusb` 看 VID/PID → 决定走 canalystii 还是 ControlCAN
2. 安装包里找 `libcontrolcan.so`（Linux）→ 有就走路线 A，没有就只能走 B
3. demo 源码里找 `VCI_OpenDevice` 的第一个参数 → 那就是 `DeviceType` 的正确值
4. 单通道自发自收（`Mode = 2`）能收到自己的帧 → 驱动通了
5. 双机对连收得到 → 再查终端电阻（见第八章第 3 条）

---

## 五、经典 CAN 与 CAN FD：先看 ECU 支持哪个

本文第九章的刷写代码是**按 CAN FD 写的**（单帧最多 62 字节、`ZCAN_TransmitFD`、`CANFD_FDF` 标志位）。如果你的盒子、或者你的 ECU 只支持经典 CAN，这套代码不能直接跑。

### 5.1 谁决定用不用 FD

顺序是**从 ECU 往回推**，不是从盒子往前推：

| 环节 | 不支持 FD 的后果 |
|:--|:--|
| **ECU 的 CAN 控制器** | 收到 FD 帧会报错帧，直接通信失败 |
| **收发器** | 需要支持 FD 的型号（如 TJA1042→TJA1044、TJA1051→TJA1057） |
| **USB-CAN 盒子** | 只能发 2.0，`DataLen > 8` 被截断 |
| **MCU 的 CAN 外设** | 例如 STM32F407 的 bxCAN **只有 2.0B，没有 FD** |

注意最后一条：**F407 做 Bootloader 时，天然就是经典 CAN 场景**。用 FD 盒子反而是浪费，配一张 2.0 兼容卡刚好。

### 5.2 ISO-TP 边界数字对照

这是降级时唯一必须改对的东西。同样是 8 字节一帧 vs 64 字节一帧，PCI 之后的可用数据差很多：

| 项 | 经典 CAN（8 字节） | CAN FD（64 字节） |
|:--|:--:|:--:|
| 单帧 SF 最大数据 | **7** | 62 |
| 首帧 FF 携带数据 | **6** | 62 |
| 连续帧 CF 携带数据 | **7** | 63 |
| SF 长度编码 | `0L`（低 4 位） | ≤7 用 `0L`，>7 用 `00 LL` |
| CF 序号范围 | 0~15 | 0~15（相同） |

> 记住一组数字：**7 / 6 / 7**（经典）和 **62 / 62 / 63**（FD）。第九章代码里 `62`、`63`、`64` 这三个魔数就是从这里来的。

### 5.3 第九章代码要改的四处

| 位置 | FD 写法 | 经典 CAN 改成 |
|:--|:--|:--|
| `can_send_raw()` | `ZCAN_TransmitFD_Data` + `CANFD_FDF` | `VCI_CAN_OBJ` + `VCI_Transmit` |
| `isotp_send()` 单帧分支 | `len <= 62`，含 `00 LL` 长 SF 分支 | `len <= 7`，**删掉**长 SF 分支 |
| `isotp_send()` 首帧/连续帧 | 首帧带 62，CF 带 63 | 首帧带 6，CF 带 7 |
| `isotp_recv()` | 缓冲区 64 字节 | 缓冲区 8 字节，`expect > 6` / `> 7` |

改完之后，`buf` 数组从 `BYTE buf[64]` 缩到 `BYTE buf[8]`，其余逻辑（PCI 判断、序号回绕、流控帧）完全不动。

**别忘了一处细节**：经典 CAN 的单帧必须显式把剩余字节补 0 或按实际长度发。ZLG 老库的 `VCI_CAN_OBJ.DataLen` 是"有效字节数"，硬件只发前 `DataLen` 个；但有些 ECU 会按固定 8 字节解析，这时要把 `DataLen` 填 8。

### 5.4 速度差多少

| 场景 | 1 MB 固件耗时 |
|:--|:--|
| 经典 CAN @ 500 kbps | **3~5 分钟** |
| CAN FD @ 500 kbps / 2 Mbps（开 BRS） | **30 秒内** |

差 6~10 倍。做台架验证、一天刷十几次的时候，这个差距很明显；但如果你的 ECU 本来只有 bxCAN，就没得选——**协议上限由 ECU 决定，不是盒子**。

---

## 六、标准调用流程

不管是哪代库，顺序都是固定的，**乱序必失败**：

```
打开设备 → 初始化通道(设波特率/滤波/模式) → 启动通道 → 收发循环 → 关闭设备
```

zlgcan 版骨架：

```c
#include "zlgcan.h"

DEVICE_HANDLE  dev;
CHANNEL_HANDLE chn;
ZCAN_INIT      init = {0};

dev = ZCAN_OpenDevice(ZCAN_USBCANFD_200U, 0, 0);   // 第 0 个设备
if (dev == 0) { /* 打开失败：查驱动/权限/设备是否被占用 */ }

ZCAN_SetValue(dev, "0", "1");                       // 开通道0终端电阻
ZCAN_SetValue(dev, "canfd_abit_baud_rate", "500000");
ZCAN_SetValue(dev, "canfd_dbit_baud_rate", "2000000");
ZCAN_SetValue(dev, "canfd_standard", "0");          // ISO

init.acc_code = 0; init.acc_mask = 0xFFFFFFFF;
init.filter = 1;   init.mode = 0;
chn = ZCAN_InitCAN(dev, 0, &init);                  // 通道索引从 0 开始
if (chn == 0) { /* 初始化失败 */ }

ZCAN_StartCAN(chn);

/* ---- 发 ---- */
ZCAN_TransmitFD_Data tx = {0};
tx.transmit_type     = 0;
tx.frame.can_id      = 0x123;
tx.frame.can_dlc     = 8;
tx.frame.flags       = CANFD_FDF | CANFD_BRS;
for (int i = 0; i < 8; i++) tx.frame.data[i] = (BYTE)i;
ZCAN_TransmitFD(chn, &tx, 1);

/* ---- 收 ---- */
ZCAN_ReceiveFD_Data rx[100];
UINT n = ZCAN_ReceiveFD(chn, rx, 100, 100);   // wait_time = 100 ms
for (UINT i = 0; i < n; i++) {
    /* rx[i].frame.can_id / can_dlc / data[] */
}

ZCAN_CloseDevice(dev);
```

老库骨架差异只在：`VCI_OpenDevice(DEV_TYPE, 0, 0)` → `VCI_InitCAN(..., &cfg)` → `VCI_StartCAN(...)` → `VCI_Transmit/Receive(..., &obj, len)`，全程重复传 `(DEV_TYPE, DeviceInd, CANInd)`。

**索引约定**：设备索引 `DeviceInd` 从 0 开始（同型号多台按插入/序列号排序），通道 `CANInd` 也从 0 开始——**两个都从 0 起**，别把 CAN1 写成 1 之后又减一次。

---

## 七、Python 侧接入

不想写 C 的话，最省事的是 `python-can` 的官方 `zlgcan` 接口（底层是 Rust 封装的 `zlgcan-driver`）：

```bash
pip install zlgcan
```

```python
import can
from zlgcan import ZCANDeviceType

bus = can.Bus(
    interface="zlgcan",
    libpath="./library",                       # 官方 DLL/SO 所在目录
    device_type=ZCANDeviceType.ZCAN_USBCANFD_200U,
    configs=[{"bitrate": 500000, "dbitrate": 2000000, "resistance": 1}],
)

# 发一帧
bus.send(can.Message(arbitration_id=0x123, is_extended_id=False,
                     data=[1, 2, 3, 4, 5, 6, 7, 8], is_fd=True, bitrate_switch=True))

# 收
msg = bus.recv(timeout=1.0)
print(hex(msg.arbitration_id), msg.data)

bus.shutdown()
```

要点：

- `configs` 是**每通道一个 dict**，双通道设备就传两个，`bitrate` 必填。
- 带 `dbitrate` 表示按 CAN FD 开通道（`chl_type` 自动选 `CANFD_ISO`）。
- `libpath` 必须指向厂商提供的 `library/` 目录，否则报"找不到动态库"。
- Linux 下如果 `libzlgcan.so` 放系统路径，`libpath` 可以留空。

---

## 八、常见坑

1. **用 ControlCAN 驱动 CAN FD 设备** → 只能跑 2.0，`DataLen > 8` 直接被截断。FD 必须换 `zlgcan`。
2. **返回值判断反了** → ZLG 全系 `1 = 成功`、`0 = 失败`，不是 0 成功。
3. **漏开终端电阻** → 两端设备对连、总线短距离时常常收不到帧，`ZCAN_SetValue(dev, "0", "1")` 先补上。总线两端各 120 Ω，**不要每台都开**。
4. **索引从 1 开始写** → `DeviceInd` / `CANInd` 都从 **0** 起。
5. **忘记 `StartCAN`** → `InitCAN` 只是配置，不启动收不到任何帧。
6. **同一设备被两个进程打开** → 第二个 `OpenDevice` 返回 0。调试软件（ZCANPRO）没关也是这个现象。
7. **32/64 位库不匹配** → Python 是 64 位就必须配 64 位 DLL/SO，否则 `OSError: 找不到模块` 或直接崩。
8. **Linux 无权限** → USB 设备需要 udev 规则或 root；否则 `OpenDevice` 静默失败。
9. **时间戳单位搞错** → ControlCAN 的 `TimeStamp` 是 **0.1 ms**，不是 ms 也不是 us。
10. **只听模式却在发帧** → `Mode = 1`（只听）下发送无效且不报错，排查时容易误认为硬件坏了。

---

## 九、用 ZLG 盒子做 UDS 刷写：核心实现

> 这一节是基于上面 zlgcan 接口的**实战落地**：用周立功 USB-CANFD 盒子当"诊断口"，通过 ISO 15765-2（CAN 上的分包传输，俗称 ISO-TP）+ ISO 14229（UDS 应用层）给 ECU 刷程序。只给核心代码骨架，不做全量封装。
>
> ⚠️ **下面 9.4–9.6 的代码是 CAN FD 版**（单帧 62 字节、`ZCAN_TransmitFD`）。用经典 CAN 盒子和只支持 2.0 的 ECU（比如 STM32F407 的 bxCAN）时，必须先按**第五章**改成 7/6/7 的边界数字和 `VCI_CAN_OBJ` 收发。协议逻辑本身完全不变，改的只是边界和收发函数。

### 9.1 协议栈分层

```
  应用层    UDS (ISO 14229)      0x10 会话 / 0x27 解锁 / 0x34-0x37 刷写
  ---------- --------------------------------------------------------
  传输层    ISO-TP (ISO 15765-2) SF / FF / CF / FC 四种帧
  ---------- --------------------------------------------------------
  数据链路  CAN / CAN FD          ZLG 盒子跑这一层
```

**一次 UDS 刷写的完整时序**（典型 29 位寻址，物理请求 `0x18DA<ECU>F1`、响应 `0x18DAF1<ECU>`）：

```
1. 进入扩展会话      0x10 0x03
2. 关闭普通通信      0x28 0x03 0x01        （只收不收，刷写期间防止干扰）
3. 安全访问          0x27 0x01 → 0x27 0x02 <seed→key>
4. 写指纹（可选）    0x2E 0xF1 0x84 <VIN/日期/装机次数>
5. 预编程条件检查    0x31 0x01 0xFF 0x00  （例程控制，扩展会话前置）
6. 擦除 Flash        0x31 0x01 0xFF 0x00 0xFF 0x00 <memoryAddress+size>
7. 请求下载          0x34 <dataFormat> <addrLen> <addr> <size>
8. 传输数据          0x36 <blockSeq> <data>  （循环，每块 ≤ 4095 字节）
9. 退出传输          0x37
10. 校验（可选）     0x31 0x01 0x02 0x02
11. 复位             0x11 0x01
```

### 9.2 ISO-TP 帧格式

标准寻址（11 位 ID）每帧 8 字节，首字节是 PCI：

| 类型 | PCI 高 4 位 | 格式 | 说明 |
|:--|:--|:--|:--|
| **SF** 单帧 | `0` | `0L DD...` | L = 长度 1~7，一帧传完 |
| **FF** 首帧 | `1` | `1L LL DD DD...` | 总长度 12 位（LLL），后 6 字节是数据 |
| **CF** 连续帧 | `2` | `2N DD...` | N = 序号 0~15，循环 |
| **FC** 流控 | `3` | `3S BS ST` | S=0继续/1等待/2溢出，BS=块大小，ST=最小间隔 ms |

CAN FD 的 SF 用 PCI 低 4 位 + 第二字节组合表示长度（`00 LL`），最大 62 字节数据，其余规则相同。

### 9.3 UDS 服务码速查

| 服务 | SID | 常用子功能 |
|:--|:--|:--|
| 诊断会话控制 | `0x10` | 01 默认 / 02 编程 / 03 扩展 |
| ECU 复位 | `0x11` | 01 硬复位 / 03 软复位 |
| 安全访问 | `0x27` | 01 请求种子 / 02 发送密钥 |
| 通信控制 | `0x28` | 00 使能 / 03 禁收发 |
| 写数据 | `0x2E` | DID（2 字节） |
| 例程控制 | `0x31` | 01 启动 / 02 停止 / 03 查询结果 |
| 请求下载 | `0x34` | 数据格式 + 地址 + 长度 |
| 传输数据 | `0x36` | 块序号 + 数据 |
| 退出传输 | `0x37` | — |
| 否定响应 | `0x7F` | 见 9.7 NRC 表 |

**肯定响应 = SID + 0x40**（`0x10` → `0x50`，`0x34` → `0x74`）。

### 9.4 核心代码：ISO-TP 发送（多帧分包）

```c
/* ---------- 常量 ---------- */
#define UDS_REQ_ID   0x18DA10F1UL   /* 物理请求：Tester → ECU */
#define UDS_RESP_ID  0x18DAF110UL   /* 物理响应：ECU → Tester */
#define ISOTP_STMIN  0              /* 最小帧间隔 ms，按 ECU 的 FC 动态改 */
#define ISOTP_BS     0              /* 块大小，0 = 不限 */

static CHANNEL_HANDLE g_chn;

static int can_send_raw(UINT id, const BYTE *d, UINT len)
{
    ZCAN_TransmitFD_Data tx = {0};
    tx.transmit_type  = 0;                 /* 正常发送 */
    tx.frame.can_id   = id;
    tx.frame.can_dlc  = len;
    tx.frame.flags    = CANFD_FDF;         /* 走 FD；纯 CAN2.0 用 ZCAN_Transmit */
    memcpy(tx.frame.data, d, len);
    return ZCAN_TransmitFD(g_chn, &tx, 1) == 1 ? 0 : -1;
}

/* 发送一条完整 UDS 报文：自动选 SF / FF+CF（CAN FD 版）*/
int isotp_send(const BYTE *data, UINT len)
{
    BYTE buf[64];

    if (len <= 62) {                        /* ---- 单帧 ---- */
        memset(buf, 0, 64);
        if (len <= 7) {                     /* 短 SF：0L + 数据 */
            buf[0] = len & 0x07;
            memcpy(&buf[1], data, len);
            return can_send_raw(UDS_REQ_ID, buf, len + 1);
        } else {                            /* 长 SF：00 LL + 数据（FD 专用）*/
            buf[0] = 0x00;
            buf[1] = len & 0xFF;
            memcpy(&buf[2], data, len);
            return can_send_raw(UDS_REQ_ID, buf, len + 2);
        }
    }

    /* ---- 首帧 ---- */
    memset(buf, 0, 64);
    buf[0] = 0x10 | ((len >> 8) & 0x0F);
    buf[1] = len & 0xFF;
    memcpy(&buf[2], data, 62);
    if (can_send_raw(UDS_REQ_ID, buf, 64)) return -1;

    UINT  off  = 62;
    BYTE  seq  = 1;                          /* CF 序号从 1 开始，回绕 0~15 */
    while (off < len) {
        UINT n = (len - off) > 63 ? 63 : (len - off);
        memset(buf, 0, 64);
        buf[0] = 0x20 | (seq & 0x0F);
        memcpy(&buf[1], data + off, n);
        if (can_send_raw(UDS_REQ_ID, buf, n + 1)) return -1;
        off += n;
        seq  = (seq + 1) & 0x0F;
        if (ISOTP_STMIN) usleep(ISOTP_STMIN * 1000);
    }
    return 0;
}
```

> 经典 CAN（8 字节）版本：SF 最大 7 字节、FF 后 6 字节、CF 每帧 7 字节，改一下边界数字即可。

### 9.5 核心代码：ISO-TP 接收（拆包）

```c
/* 收一条完整 UDS 响应，超时 timeout_ms */
int isotp_recv(BYTE *out, UINT *out_len, int timeout_ms)
{
    ZCAN_ReceiveFD_Data rx[64];
    UINT total = 0, expect = 0;
    BYTE seq = 1;
    DWORD t0 = GetTickCount();   /* Linux 换成 clock_gettime(CLOCK_MONOTONIC) */

    while ((int)(GetTickCount() - t0) < timeout_ms) {
        UINT n = ZCAN_ReceiveFD(g_chn, rx, 64, 20);
        for (UINT i = 0; i < n; i++) {
            if ((rx[i].frame.can_id & 0x1FFFFFFF) != UDS_RESP_ID) continue;

            BYTE *d = rx[i].frame.data;
            UINT  dl = rx[i].frame.can_dlc;
            BYTE  pci = d[0] & 0xF0;

            if (pci == 0x00) {                       /* SF */
                UINT L, off;
                if ((d[0] & 0x0F) == 0) {            /* 长 SF：00 LL */
                    L = d[1];        off = 2;
                } else {                             /* 短 SF：0L */
                    L = d[0] & 0x07; off = 1;
                }
                memcpy(out, &d[off], L);
                *out_len = L;
                return 0;
            } else if (pci == 0x10) {                /* FF */
                expect = ((d[0] & 0x0F) << 8) | d[1];
                UINT c = (expect > 62) ? 62 : expect;
                memcpy(out, &d[2], c);
                total = c;
                seq = 1;
                /* 回流控帧：继续 / 不限块 / 无最小间隔 */
                BYTE fc[3] = { 0x30, 0x00, 0x00 };
                can_send_raw(UDS_REQ_ID, fc, 3);
            } else if (pci == 0x20) {                /* CF */
                if ((d[0] & 0x0F) != seq) return -2;  /* 序号错乱 */
                UINT c = (expect - total) > 63 ? 63 : (expect - total);
                memcpy(out + total, &d[1], c);
                total += c;
                seq = (seq + 1) & 0x0F;
                if (total >= expect) { *out_len = expect; return 0; }
            }
        }
    }
    return -1;   /* 超时 */
}
```

### 9.6 核心代码：刷写主流程

```c
/* 发一条 UDS 请求并等肯定响应；返回 0 成功 */
static int uds_request(const BYTE *req, UINT len, BYTE *resp, UINT *rlen)
{
    if (isotp_send(req, len)) return -1;
    if (isotp_recv(resp, rlen, 1000)) return -2;

    if (resp[0] == 0x7F) {                  /* 否定响应 */
        printf("NRC: 0x%02X (service 0x%02X)\n", resp[2], resp[1]);
        return -3;
    }
    if (resp[0] != (req[0] + 0x40)) return -4;  /* 响应码不匹配 */
    return 0;
}

/* 一条请求的便捷包装 */
#define UDS_REQ(...) do {                                        \
        BYTE _r[64]; UINT _rl;                                   \
        BYTE _q[] = { __VA_ARGS__ };                             \
        if (uds_request(_q, sizeof(_q), _r, &_rl)) goto fail;     \
    } while (0)

int uds_flash(const BYTE *fw, UINT fw_len, UINT addr)
{
    BYTE  resp[64]; UINT rlen;
    BYTE  req[16];

    /* 1. 扩展会话 */
    UDS_REQ(0x10, 0x03);

    /* 2. 关普通通信（可选，防干扰） */
    UDS_REQ(0x28, 0x03, 0x01);

    /* 3. 安全访问：请求种子 → 算密钥 → 发送 */
    UDS_REQ(0x27, 0x01);
    /* resp[2..5] 是 4 字节种子，算法由 ECU 厂定义 */
    {
        BYTE seed[4] = { resp[2], resp[3], resp[4], resp[5] };
        BYTE key[4];
        security_algo(seed, key);            /* ← 换成你的算法 */
        UDS_REQ(0x27, 0x02, key[0], key[1], key[2], key[3]);
    }

    /* 4. 进入编程会话 */
    UDS_REQ(0x10, 0x02);

    /* 5. 请求下载：0x34 + 格式 + 地址长度 + 地址 + 长度 */
    req[0] = 0x34; req[1] = 0x00;  req[2] = 0x44;   /* 00 无压缩无加密, 44 = 4字节地址+4字节长度 */
    req[3] = (addr >> 24) & 0xFF; req[4] = (addr >> 16) & 0xFF;
    req[5] = (addr >>  8) & 0xFF; req[6] =  addr        & 0xFF;
    req[7] = (fw_len >> 24) & 0xFF; req[8] = (fw_len >> 16) & 0xFF;
    req[9] = (fw_len >>  8) & 0xFF; req[10] = fw_len       & 0xFF;
    if (uds_request(req, 11, resp, &rlen)) goto fail;

    UINT max_block = ((resp[2] & 0x0F) << 8) | resp[3];   /* ECU 允许的最大块 */
    if (max_block == 0 || max_block > 4095) max_block = 1024;

    /* 6. 循环传输数据 */
    BYTE seq = 1;
    for (UINT off = 0; off < fw_len; ) {
        UINT n = (fw_len - off) > max_block ? max_block : (fw_len - off);
        BYTE blk[4096 + 2];
        blk[0] = 0x36;
        blk[1] = seq;
        memcpy(&blk[2], fw + off, n);
        if (uds_request(blk, n + 2, resp, &rlen)) goto fail;

        off += n;
        seq  = (seq == 0xFF) ? 0x00 : seq + 1;   /* 0x36 的序号 0~255，进位不跳号 */
        printf("\r%d/%d  %d%%", off, fw_len, off * 100 / fw_len);
    }

    /* 7. 退出传输 */
    UDS_REQ(0x37);

    /* 8. 校验（部分 ECU 支持） */
    /* UDS_REQ(0x31, 0x01, 0x02, 0x02); */

    /* 9. 复位 */
    UDS_REQ(0x11, 0x01);
    return 0;

fail:
    return -1;
}
```

### 9.7 NRC 否定响应码

| NRC | 含义 | 常见原因 |
|:--|:--|:--|
| `0x10` | 一般拒绝 | 当前条件不满足 |
| `0x11` | 服务不支持 | SID 打错 |
| `0x12` | 子功能不支持 | 子功能或格式不对 |
| `0x13` | 长度错误 | 参数长度不符 |
| `0x22` | 条件不满足 | 未进正确会话 / 未解锁 |
| `0x24` | 请求序列错 | 漏了前置步骤（如没擦除就下载） |
| `0x31` | 地址/长度越界 | 刷写地址不在 Flash 区 |
| `0x33` | 安全访问被拒 | 密钥错 / 尝试次数超限 |
| `0x35` | 密钥无效 | 算法不对 |
| `0x36` | 超过尝试次数 | 锁定，需等待或重新上电 |
| `0x72` | 编程失败 | Flash 写入错 |
| `0x73` | 块序号错 | 0x36 的 blockSequenceCounter 跳了 |
| `0x78` | 响应挂起 | **不是错误**，ECU 还在处理，需继续等 |

> `0x78` 要特别注意：ECU 擦 Flash 时常返回 `7F 31 78`，表示"收到，等我"。此时**必须继续等**下一个响应（典型 P2* 超时 5 s），不能当失败重发。

### 9.8 几个必须知道的坑

1. **`0x36` 的序号和 ISO-TP 的 CF 序号是两套**。`0x36` 的 blockSequenceCounter 是 **0x00~0xFF**，到 FF 回 00；ISO-TP 的 CF 序号是 **0~15**，到 15 回 0。两个独立，别混淆。
2. **CAN FD 刷写快很多**。经典 CAN 500 kbps 刷 1 MB 约 3~5 分钟；CAN FD 500 k/2 M 能压到 30 秒内。如果 ECU 支持 FD，一定开 **BRS**。
3. **流控帧（FC）**。严格实现里，收到 FF 后 tester 必须回 `30 00 00`（继续，不限块，无间隔）。很多 ECU 不强求，但不回某些 ECU 会卡死。
4. **P2 / P2\* 超时**。P2 默认 50 ms，收到 `0x78` 后改用 P2\*（默认 5000 ms）。自己写工具时 timeout 至少要给 5 s。
5. **擦除比传输慢**。`0x31 01 FF00` 擦 Flash 常要 1~10 秒，期间会发 `0x78`。串口超时设短了就会误判失败。
6. **安全访问次数限制**。连续 3 次密钥错 ECU 会锁（NRC 0x36），要重新上电或等定时器。调试算法时先在台架上试，别拿真车试。
7. **物理寻址 vs 功能寻址**。物理 `0x18DA10F1` 一对一定向；功能 `0x18DB33F1` 广播（只能用于 `0x10`/`0x3E` 这类不要求唯一响应的服务）。刷写**必须用物理寻址**。
8. **`0x28` 关闭通信后记得恢复**。刷写完 ECU 复位会自动恢复，但如果中途失败，ECU 可能停留在"只发不收"状态，需要断电。
9. **Flash 驱动要配等待周期**。168 MHz 下 F407 的 Flash 需要 WS=5，否则写入必错——这条回 RM0090 第 3 章查。
10. **别在扩展会话里跑应用逻辑**。刷写会话专属，刷完必须 `0x11 01` 复位回默认会话。

---

## 十、一页速查

| 阶段 | ControlCAN | zlgcan |
|:--|:--|:--|
| 打开 | `VCI_OpenDevice(type, ind, 0)` | `ZCAN_OpenDevice(type, ind, 0)` → 句柄 |
| 初始化 | `VCI_InitCAN(type, ind, ch, &cfg)` | `ZCAN_InitCAN(dev, ch, &init)` → 句柄 |
| 启动 | `VCI_StartCAN(type, ind, ch)` | `ZCAN_StartCAN(chn)` |
| 发送 | `VCI_Transmit(type, ind, ch, &obj, n)` | `ZCAN_Transmit / ZCAN_TransmitFD(chn, &tx, n)` |
| 接收 | `VCI_Receive(..., &obj, n, wait)` | `ZCAN_Receive / ZCAN_ReceiveFD(chn, &rx, n, wait)` |
| 关闭 | `VCI_CloseDevice(type, ind)` | `ZCAN_CloseDevice(dev)` |
| 成功判定 | `== STATUS_OK (1)` | `== 1` |

> 记住一句话：**句柄化的是新库，三元组的是老库；FD 只在新库；成功都是 1。**

### 10.1 三条路的选型

| 你的设备 | 走哪条路 | 参考章节 |
|:--|:--|:--|
| ZLG 原厂 + 要 CAN FD | `zlgcan`（`ZCAN_*`） | 三、六、九 |
| ZLG 原厂 USB-CAN II 等老设备 | `ControlCAN`（`VCI_*`） | 二、六 |
| 创芯科技 / 吉阳等兼容卡 | `ControlCAN` 或 python-can `canalystii` 后端 | 四 |
| 只支持 CAN 2.0 的 ECU（如 F407 bxCAN） | 任选盒子，代码按经典 CAN 改 | 五 |

> 最后提醒：**协议上限由 ECU 决定，不由盒子决定**。ECU 只有 bxCAN 时，买 FD 盒子也不会更快；反过来，盒子只支持 2.0 时，ECU 再强也只能按 3~5 分钟刷 1 MB 的速度走。