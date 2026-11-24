---
{"dg-publish":true,"permalink":"/knowledge/embedded/04-ESP32-CAM入门与图传实战/","title":"ESP32-CAM 入门与图传实战","tags":["ESP32-CAM","OV2640","摄像头","图传","MJPEG","SD卡","Arduino","嵌入式","监控"],"dg-note-properties":{"title":"ESP32-CAM 入门与图传实战","date":"2026-11-14","tags":["ESP32-CAM","OV2640","摄像头","图传","MJPEG","SD卡","Arduino","嵌入式","监控"]}}
---


# 📷 ESP32-CAM 入门与图传实战

> 一块 20 元的带摄像头小板子，能做什么：局域网实时图传、定时拍照存 SD 卡、远程监控、机器人的「眼睛」。
> 但它的 GPIO 几乎被摄像头吃光，定位是**带视觉的传感器节点**，不是通用控制板。

---

## 📑 本页导航

| 章节 | 内容 | 难度 |
|:--|:--|:--|
| 一、认识 ESP32-CAM | 规格、引脚占用、能干什么不能干什么 | ⭐ |
| 二、接线与烧录 | USB-TTL、IO0 下载模式、供电陷阱 | ⭐⭐ |
| 三、跑通官方例程 | CameraWebServer、参数调节 | ⭐⭐ |
| 四、自己写图传 | camera_config 逐项说明、分辨率与帧率 | ⭐⭐⭐ |
| 五、拍照存 SD 卡 | SD_MMC 挂载、帧缓冲归还 | ⭐⭐⭐ |
| 六、无 PSRAM 的应对 | 分辨率和内存的现实 | ⭐⭐⭐ |
| 七、排障手册 | 按症状查 | — |
| 八、坑位汇总 | 一次性避坑 | — |

---

# 一、认识 ESP32-CAM

## 1.1 关键规格

| 项 | 值 |
|:--|:--|
| 芯片 | ESP32-S（**双核** 240MHz）|
| 内存 | 520KB SRAM + **4MB PSRAM**（做 JPEG 缓冲必需）|
| WiFi | 2.4GHz + **蓝牙 4.2** |
| 摄像头 | **OV2640**，200 万像素，8 位并行 DVP 接口 |
| 可用 GPIO | 只剩 12 / 13 / 14 / 15 / 16（16 还被 SD 占用）|
| 板载 USB | ❌ **无**，必须外接 USB-TTL |
| 烧录方式 | 手动接线 + IO0 接地进下载模式 |
| 价格 | ~20 元（含摄像头）|

**双核的意义**：一个核跑 WiFi 协议栈，一个核跑你的代码。所以图传时不会因为处理逻辑卡住网络。

## 1.2 引脚映射（AI Thinker 板）

| 功能 | GPIO | | 功能 | GPIO |
|:--|:--|:--|:--|:--|
| PWDN | 32 | | D0 (Y2) | 5 |
| RESET | -1（未接）| | D1 (Y3) | 18 |
| XCLK | 0 | | D2 (Y4) | 19 |
| SIOD (SDA) | 26 | | D3 (Y5) | 21 |
| SIOC (SCL) | 27 | | D4 (Y6) | 36 |
| D5 (Y7) | 39 | | D6 (Y8) | 34 |
| VSYNC | 25 | | D7 (Y9) | 35 |
| HREF | 23 | | PCLK | 22 |
| 闪光灯 LED | 4 | | SD 卡 | 2 / 14 / 15 |
| D7 (Y9) 复用 | 16 | | | |

> ⚠️ **可用 GPIO 只剩 12、13、14、15、16**，其中 16 还被 SD 卡占用。这就是为什么 ESP32-CAM 不适合当通用控制板 —— 想驱动电机、舵机，请交给 ESP8266 或 ESP32 标准板。

## 1.3 它能干什么 / 不能干什么

| ✅ 擅长 | ❌ 不适合 |
|:--|:--|
| 局域网 MJPEG 实时图传 | 驱动电机、舵机（引脚不够）|
| 定时/触发拍照存 SD 卡 | 高帧率高清视频（WiFi 带宽不够）|
| 简单运动检测（帧差分）| 复杂图像识别（算力有限）|
| 机器人视觉传感器 | 长时间高负载运行（发热严重）|

---

# 二、接线与烧录

## 2.1 接线（USB-TTL）

ESP32-CAM 没有 USB 口，用 CH340 / CP2102 模块：

```
USB-TTL          ESP32-CAM
 GND  ───────────  GND        ← 必须共地
 5V   ───────────  5V         ← 用 5V，3.3V 供电不足
 TX   ───────────  U0R (RX)
 RX   ───────────  U0T (TX)
```

> ⚠️ **供电是最大的坑**：ESP32-CAM 启动瞬间电流可达 300mA+，USB-TTL 的 3.3V 针脚给不起。务必接 **5V**（板上有 LDO）。

## 2.2 烧录流程

没做好会一直 `Failed to connect`：

1. **IO0 接 GND**（进入下载模式）
2. 按一下 **RST**（或断开再插 5V）
3. Arduino IDE 点上传
4. 看到 `Connecting...` 转圈后出现 `Writing at 0x0000...` 就是通了
5. 烧完**拔掉 IO0 的 GND**，再按一次 RST

## 2.3 开发板设置

| 项 | 值 |
|:--|:--|
| Board | `AI Thinker ESP32-CAM` |
| PSRAM | **Enabled**（关键！）|
| Partition Scheme | `Huge APP (3MB No OTA/1MB SPIFFS)` |
| Upload Speed | 115200（不稳降 57600）|

> 💡 Board 列表里找不到 AI Thinker ESP32-CAM，说明 ESP32 板卡包太旧，升级到 2.x 以上。

## 2.4 主机环境依赖（PlatformIO）

ESP32-CAM 没有板载 USB 转串口，但**主机侧的环境要求与 ESP8266 基本一致**。以下版本在 Ubuntu 20.04.6 上实测可用。

### 需要装什么

| 组件 | 版本 | 说明 |
|:--|:--|:--|
| PlatformIO Core | 6.1.19 | `pio` 命令本体 |
| 平台包 | espressif32 @ 8.0.0 | 板卡定义与构建规则，含 314 个板 |
| Arduino 框架 | framework-arduinoespressif32 @ 4.30312.261116 | 对应 Arduino core 3.3.12 |
| 预编译库 | framework-arduinoespressif32-esp32-libs @ 3.3.12 | 含相机驱动，约 184 MB |
| 编译器（Xtensa）| toolchain-xtensa-esp-elf @ 14.2.0 | ESP32 / S2 / S3 |
| 编译器（RISC-V）| toolchain-riscv32-esp @ 14.2.0 | C3 / C6 / H2 |

```bash
pio --version        # PlatformIO Core, version 6.1.19
pio pkg list -g
```

### 相机驱动不用另外装

这是最容易走弯的一步：`esp_camera.h` 已经随 `esp32-libs` 装好了，**不要再往 `lib_deps` 里加 `esp32-camera`**，否则会与内置版本冲突。

```
framework-arduinoespressif32-esp32-libs/
  include/espressif__esp32-camera/driver/include/esp_camera.h
  lib/libespressif__esp32-camera.a        ← 已预编译
```

同理，`framework-espidf` 也不用装：本例走 Arduino 框架，完全不碰 IDF。

### 串口权限

与 ESP8266 完全相同，装 PlatformIO 官方 udev 规则最省事（直接给 `0666`）：

```bash
curl -fsSL -o /tmp/99-platformio-udev.rules \
  https://raw.githubusercontent.com/platformio/platformio-core/develop/platformio/assets/system/99-platformio-udev.rules

sudo cp /tmp/99-platformio-udev.rules /etc/udev/rules.d/
sudo chown root:root /etc/udev/rules.d/99-platformio-udev.rules
sudo chmod 644 /etc/udev/rules.d/99-platformio-udev.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

ESP32-CAM 用 CP2102 或 CH340 转接板（`10c4:ea60` / `1a86:7523`），都在规则覆盖范围内。

### 串口工具

```bash
sudo apt-get install -y picocom
picocom -b 115200 /dev/ttyUSB0     # 退出：Ctrl-A 然后 Ctrl-X
```

看启动日志时特别有用：ESP32-CAM 上电会打印分区表、PSRAM 大小、`Brownout` 报警，`picocom` 逐行显示不会丢。

### 最小工程

```ini
; platformio.ini
[env:esp32cam]
platform = espressif32
board = esp32cam
framework = arduino
monitor_speed = 115200
upload_speed = 460800
board_build.flash_mode = dio
```

```bash
pio run                    # 编译
pio run -t upload          # 烧录
pio device monitor         # 看串口，Ctrl-] 退出
```

### 板卡定义里已经配好的项

`board = esp32cam` 不是空壳，它替你设了这些，**不要手动覆盖**：

| 项 | 值 | 作用 |
|:--|:--|:--|
| `-DBOARD_HAS_PSRAM` | 已开启 | 不写这行，SVGA 以上会失败 |
| `-mfix-esp32-psram-cache-issue` | 已开启 | 规避 ESP32 PSRAM 的 cache bug |
| `partitions` | `huge_app.csv` | app 3MB，能装下 CameraWebServer |
| `flash_mode` | `dio` | 板上 flash 就是 DIO 接法 |

⚠️ 如果自己改分区表，把 app 分区做到 2MB 以下，会撞 `text section exceeds available space in board`。

### 两条硬件侧的限制（软件补不了）

**没有板载 USB。** 必须外接 USB-TTL 转接板，且 PlatformIO 的自动复位对这块板无效（DTR/RTS 没接到 EN/IO0）。烧录时需手动把 GPIO0 接 GND 再复位；串口会停在 `Connecting........`。

**供电必须给足。** 摄像头启动瞬间 300mA+，WiFi 发射再加 200mA，USB-TTL 的 5V 引脚常常供不住，表现是反复 `Brownout detector was triggered`。用外部 5V 或单独 LDO，并在 5V/GND 间并 100µF 以上电容。

---

# 三、跑通官方例程

`文件 → 示例 → ESP32 → Camera → CameraWebServer`

改两处：

```cpp
// 1) 选对型号
#define CAMERA_MODEL_AI_THINKER   // 取消注释

// 2) 填 WiFi（只能 2.4GHz）
const char* ssid = "你的WiFi";
const char *password = "你的密码";
```

烧完串口打印：

```
Camera Ready! Use 'http://192.168.1.123' to connect
```

浏览器打开这个地址，能看到实时画面面板：可调分辨率、亮度、对比度、特效（黑白/负片）、镜像翻转。

**这个例程值得读的地方**：Web 服务器怎么用 chunked 方式连续推 JPEG 帧、`/capture` 拍照接口怎么写、参数怎么用 query string 改。读懂它，自己写图传就很简单。

---

# 四、自己写图传：camera_config 详解

## 4.1 最小初始化代码

```cpp
#include "esp_camera.h"
#include "camera_pins.h"    // 在 CameraWebServer 例程目录里，可拷过来

void setup() {
  Serial.begin(115200);

  camera_config_t config = {};
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer   = LEDC_TIMER_0;

  // 数据线 D0~D7
  config.pin_d0 = Y2_GPIO_NUM;   // 5
  config.pin_d1 = Y3_GPIO_NUM;   // 18
  config.pin_d2 = Y4_GPIO_NUM;   // 19
  config.pin_d3 = Y5_GPIO_NUM;   // 21
  config.pin_d4 = Y6_GPIO_NUM;   // 36
  config.pin_d5 = Y7_GPIO_NUM;   // 39
  config.pin_d6 = Y8_GPIO_NUM;   // 34
  config.pin_d7 = Y9_GPIO_NUM;   // 35

  // 控制线
  config.pin_xclk     = XCLK_GPIO_NUM;   // 0
  config.pin_pclk     = PCLK_GPIO_NUM;   // 22
  config.pin_vsync    = VSYNC_GPIO_NUM;  // 25
  config.pin_href     = HREF_GPIO_NUM;   // 23
  config.pin_sccb_sda = SIOD_GPIO_NUM;   // 26
  config.pin_sccb_scl = SIOC_GPIO_NUM;   // 27
  config.pin_pwdn     = PWDN_GPIO_NUM;   // 32
  config.pin_reset    = RESET_GPIO_NUM;  // -1

  config.xclk_freq_hz = 20000000;        // 20MHz，别再高
  config.pixel_format = PIXFORMAT_JPEG;  // 图传/存卡都用 JPEG
  config.grab_mode    = CAMERA_GRAB_LATEST;
  config.fb_location  = CAMERA_FB_IN_PSRAM;

  if (psramFound()) {
    config.frame_size   = FRAMESIZE_SVGA;  // 800x600
    config.jpeg_quality = 12;              // 0~63，越小越好越小画面越大
    config.fb_count     = 2;               // 双缓冲，帧率更高
  } else {
    config.frame_size   = FRAMESIZE_QVGA;  // 320x240
    config.jpeg_quality = 15;
    config.fb_count     = 1;
    config.fb_location  = CAMERA_FB_IN_DRAM;
  }

  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Camera init failed: 0x%x\n", err);
    return;
  }
  Serial.println("Camera ready");
}
```

## 4.2 关键参数怎么调

| 参数 | 建议 | 说明 |
|:--|:--|:--|
| `pixel_format` | `PIXFORMAT_JPEG` | 图传输送、存卡省空间；要做图像处理才用 RGB565/GRAYSCALE |
| `frame_size` | SVGA 图传 / UXGA 拍照 | 见下表 |
| `jpeg_quality` | 10~15 | **数字越小画质越好、体积越大**，别理解反了 |
| `fb_count` | 2（有 PSRAM）| 双缓冲显著提帧率；内存紧就设 1 |
| `grab_mode` | `CAMERA_GRAB_LATEST` | 图传要「最新帧」，选它避免延迟累积 |
| `xclk_freq_hz` | 20000000 | 20MHz 是稳定上限 |

## 4.3 分辨率与帧率对照

| 分辨率 | 尺寸 | PSRAM 下帧率 | 场景 |
|:--|:--|:--|:--|
| QQVGA | 160×120 | ~30fps | 极省带宽 |
| **QVGA** | 320×240 | ~25fps | 图传流畅首选 |
| **VGA** | 640×480 | ~15fps | 清晰与流畅折中 |
| SVGA | 800×600 | ~10fps | |
| XGA | 1024×768 | ~7fps | |
| UXGA | 1600×1200 | ~5fps | **只适合拍照** |

> 无 PSRAM 时以上全部打骨折，通常只能跑 QVGA，且 `fb_count` 只能 1。

## 4.4 取帧与释放

```cpp
camera_fb_t *fb = esp_camera_fb_get();
if (!fb) {
  Serial.println("拍照失败");
  return;
}
// fb->buf 是 JPEG 数据，fb->len 是长度
Serial.printf("frame: %u bytes (%dx%d)\n", fb->len, fb->width, fb->height);

esp_camera_fb_return(fb);   // ⚠️ 必须归还
```

> ⚠️ **`fb_get` 和 `fb_return` 必须成对**。漏掉几次后 `fb_get` 就返回 NULL，看起来像「摄像头坏了」，实际是缓冲池被占满。

---

# 五、拍照存 SD 卡

## 5.1 完整代码

```cpp
#include "esp_camera.h"
#include "FS.h"
#include "SD_MMC.h"

bool initSD() {
  // 第二个参数 true = 1-bit 模式（占 GPIO 少，兼容性好）
  if (!SD_MMC.begin("/sdcard", true)) {
    Serial.println("SD 卡挂载失败");
    return false;
  }
  uint8_t type = SD_MMC.cardType();
  if (type == CARD_NONE) {
    Serial.println("没插入 SD 卡");
    return false;
  }
  Serial.printf("SD OK, size: %lluMB\n", SD_MMC.cardSize() / (1024 * 1024));
  return true;
}

void takePhoto() {
  camera_fb_t *fb = esp_camera_fb_get();
  if (!fb) { Serial.println("拍照失败"); return; }

  String path = "/photo_" + String(millis()) + ".jpg";
  File file = SD_MMC.open(path.c_str(), FILE_WRITE);
  if (file) {
    file.write(fb->buf, fb->len);
    Serial.printf("已保存 %s (%u bytes)\n", path.c_str(), fb->len);
  }
  file.close();

  esp_camera_fb_return(fb);   // ⚠️ 必须归还
}
```

## 5.2 SD 卡要求

| 项 | 要求 |
|:--|:--|
| 格式 | **FAT32**（exFAT/APFS 不认）|
| 容量 | ≤ 32GB 最稳（64GB 需工具格 FAT32）|
| 速度 | Class 10 以上，否则存图卡顿 |
| 引脚 | SD 用 GPIO 2 / 14 / 15，**与闪光灯 GPIO4 无关** |

> ⚠️ SD_MMC 用 4-bit 模式会和摄像头抢引脚。**1-bit 模式**（`begin("/sdcard", true)`）是唯一稳妥选择。

---

# 六、无 PSRAM 的应对

部分廉价 ESP32-CAM 标称有 PSRAM 实际没有，或 PSRAM 坏了。判断：

```cpp
if (psramFound()) {
  Serial.printf("PSRAM: %u bytes free\n", ESP.getFreePsram());
} else {
  Serial.println("NO PSRAM!");
}
```

无 PSRAM 的现实：

| 限制 | 应对 |
|:--|:--|
| 只能 QVGA | 接受它，或换板子 |
| `fb_location` 必须 `CAMERA_FB_IN_DRAM` | 别忘了改，否则 init 失败 0x20004 |
| `fb_count` 只能 1 | 帧率约 10fps |
| 存 SD 卡容易失败 | 降低质量和分辨率 |

也可以用 PSRAM 测试加 `config.fb_location = CAMERA_FB_IN_DRAM` 强跑更高分辨率，但极易崩 —— 不建议。

---

# 七、排障手册

| 症状 | 原因 | 处理 |
|:--|:--|:--|
| 烧录一直 `Failed to connect` | IO0 没接地 / 供电不足 / 没按 RST | IO0→GND，按 RST，用 5V 供电 |
| 烧完程序不跑 | IO0 还接着 GND | **拔掉 IO0 的地线**，再按 RST |
| 串口没输出 | 波特率错 / TX/RX 接反 | 115200；TX↔U0R，RX↔U0T |
| 图像全黑 | 排线没插紧 / 供电不足 | 重新插 FPC 排线（**断电操作**），换 5V 大电流源 |
| 图像花屏/条纹 | 电源噪声 / XCLK 太高 | 加 100µF 电容；`xclk_freq_hz` 降到 10MHz |
| `Camera init failed: 0x105` | PSRAM 未启用 | Tools → PSRAM → Enabled |
| `Camera init failed: 0x20004` | fb_location 用 PSRAM 但无 PSRAM | 改 `CAMERA_FB_IN_DRAM` |
| `Camera init failed: 0x20001` | 引脚配置错 / 型号选错 | 确认 `CAMERA_MODEL_AI_THINKER` |
| 拍几张后 `fb_get` 返回 NULL | 漏了 `esp_camera_fb_return` | 每次 get 必配对 return |
| SD 挂载失败 | 格式不对 / 容量过大 / 用了 4-bit | 格 FAT32 ≤32GB，`begin(path, true)` |
| 图传卡顿掉帧 | 分辨率太高 / WiFi 信号弱 | 降到 VGA 或 QVGA，靠近路由器 |
| 运行几分钟重启 | 发热 / 供电不足 | 加散热片，独立 5V 2A 供电 |
| WiFi 连不上 | 连了 5G | 换 2.4GHz |

---

# 八、坑位汇总

**硬件**
1. ESP32-CAM 的 3.3V 针脚电流不够，**必须接 5V**
2. 启动瞬间 300mA+，供电不足表现为「一联网就重启」
3. 摄像头 FPC 排线脆弱，**插拔必须断电**
4. 板载闪光灯 LED 在 GPIO4，强光下可作补光（也可当状态灯）
5. 长时间图传发热严重，贴散热片或降帧率

**软件**
6. `esp_camera_fb_get` 必须配 `esp_camera_fb_return`
7. `Tools → PSRAM` 忘了 Enable，会一直 init 失败
8. `jpeg_quality` 越小画质越好，别以为越大越好
9. 无 PSRAM 时 `fb_location` 必须改成 DRAM
10. 烧录后忘了拔 IO0 的地线 → 程序不跑

**网络**
11. 只支持 2.4GHz
12. 图传和 SD 写卡同时做容易抢内存，建议分时
13. 多人同时看流会打满带宽，正式用要考虑只推一份到服务器中转

---

## 学习路线建议

```
第 1 天：接线烧录，跑通 CameraWebServer
第 2 天：调分辨率/画质，理解 camera_config 每个字段
第 3 天：自己写网页图传（chunked JPEG 流）
第 4 天：拍照存 SD 卡 + FAT32 格式
第 5 天：加运动检测（帧差分）或接 ESP8266 做双板机器人
```

## 参考资料

- ESP32-CAM 官方例程：Arduino IDE `Examples → ESP32 → Camera`
- ESP32 Arduino Core 文档：`docs.espressif.com/projects/arduino-esp32/`
- Random Nerd Tutorials（ESP32-CAM 系列最全）：`randomnerdtutorials.com`
- OV2640 数据手册：OmniVision 官网

---

## 相关文档

- `03-ESP8266-NodeMCU入门与联网实战` — 打好 GPIO/WiFi 基础
