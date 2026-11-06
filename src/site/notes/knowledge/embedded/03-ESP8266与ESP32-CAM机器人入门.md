---
{"dg-publish":true,"permalink":"/knowledge/embedded/03-ESP8266与ESP32-CAM机器人入门/","title":"ESP8266 与 ESP32-CAM 机器人入门","tags":["ESP8266","ESP32-CAM","NodeMCU","摄像头","WiFi","Arduino","嵌入式","机器人","MQTT","图传"],"dg-note-properties":{"title":"ESP8266 与 ESP32-CAM 机器人入门","date":"2026-11-06","tags":["ESP8266","ESP32-CAM","NodeMCU","摄像头","WiFi","Arduino","嵌入式","机器人","MQTT","图传"]}}
---


# 🤖 ESP8266 与 ESP32-CAM 机器人入门

> 从零开始用两块现成板子走完「点灯 → 联网 → 图传 → 小车」的完整路径。
> 主线思路：**先用 ESP8266 打好基础（便宜、简单、不易劝退），再上 ESP32-CAM 拿视觉能力**，最后两块合起来做一台 WiFi 摄像头小车。

---

## 📑 本页导航

| 章节 | 内容 | 难度 |
|:--|:--|:--|
| 一、硬件与环境 | 两块板子差异、CH340 驱动、三套开发方式对比 | ⭐ |
| 二、ESP8266 入门 | GPIO / PWM / ADC / WiFi | ⭐⭐ |
| 三、ESP32-CAM 进阶 | 摄像头、视频流、SD 卡拍照 | ⭐⭐⭐ |
| 四、做一台小车 | 双板协作架构、MQTT 通信、遥控小车 | ⭐⭐⭐⭐ |
| 五、排障手册 | 按症状查 | — |
| 六、坑位汇总 | 一次性避坑 | — |

---

# 一、硬件与环境

## 1.1 两块板子的定位差异

| 维度 | ESP8266 NodeMCU Lua V3 | ESP32-CAM |
|:--|:--|:--|
| 芯片 | ESP8266EX（单核 80MHz） | ESP32-S（双核 240MHz） |
| 内存 | 80KB RAM / 4MB Flash | 520KB SRAM + **4MB PSRAM** |
| WiFi | 2.4G only | 2.4G + **蓝牙 4.2** |
| GPIO | ~11 个可用 | ~10 个可用（**多数被摄像头占用**） |
| 摄像头 | ❌ 无 | ✅ OV2640（200 万像素） |
| 板载 USB | ✅ CH340 | ❌ **无**，需 USB-TTL |
| 烧录方式 | 插线即烧 | 需手动接线 + 断电重启 |
| 价格 | ~10 元 | ~20 元（带摄像头） |

**为什么先学 8266**：
1. 插上 USB 就能烧，不折腾接线
2. 引脚未被占用，GPIO 实验随便做
3. 贵在「简单」，出错概率低，适合建立信心

**ESP32-CAM 的定位**：它是**一个带视觉的传感器节点**，不是通用的开发板。 GPIO 基本被摄像头吃完了，所以它的典型角色是「拍照/传视频」，执行控制交给别的板子。

## 1.2 CH340 驱动安装

NodeMCU V3 用的 USB 转串口芯片是 **CH340**（不是常见的 CP2102）。Windows 10/11 可能不认。

```bash
# Windows：装官方驱动
# 下载 ch341ser.exe → 关闭驱动签名强制 → 安装 → 设备管理器看 COM 口

# macOS
# 新版 macOS 需从 WCH 官网下 CH34xVCPDriver.pkg
# 装完可能出现 /dev/tty.wchusbserial1420

# Linux：内核自带，即插即用
dmesg | tail -20        # 看到 ch341-uart converter now attached to ttyUSB0 就对了
ls /dev/ttyUSB*
sudo usermod -aG dialout $USER   # 避免每次都要 sudo
```

> ⚠️ **权限坑**：Linux 下不加 `dialout` 组会报 `Permission denied: '/dev/ttyUSB0'`。

**验证连接**：串口工具（Arduino IDE 自带串口监视器 / `screen` / `minicom`）打开 `115200` 波特率，按一下板子上的 RST，应该能看到启动乱码或 AT 固件输出。

## 1.3 三套开发方式对比

| 方式 | 语言 | 上手难度 | 摄像头支持 | 适合场景 |
|:--|:--|:--|:--|:--|
| **Arduino IDE / PlatformIO** | C++ | ⭐⭐ | ✅ 完善（`esp_camera` 库） | **本教程默认** |
| MicroPython | Python | ⭐ | ⚠️ ESP32-CAM 支持弱 | 纯 8266 快速原型 |
| ESP-IDF | C | ⭐⭐⭐⭐ | ✅ 原生 | 要做产品级、需要精细控制 |

**选 Arduino 的理由**：例程最多、库最全、社区问题一搜就有答案。ESP32-CAM 的官方 `CameraWebServer` 例程就是 Arduino 写的，拿来即用。

### 安装步骤

1. **Arduino IDE 2.x**（[arduino.cc](https://www.arduino.cc/en/software)）
2. **添加板管理 URL**：
   `文件 → 首选项 → 附加开发板管理器网址`：
   ```
   https://espressif.github.io/arduino-esp32/package_esp32_index.json
   http://arduino.esp8266.com/stable/package_esp8266com_index.json
   ```
3. **开发板管理器**搜 `esp8266` 和 `esp32`，各自安装
4. **选板子**：
   - 8266 → `Tools → Board → NodeMCU 1.0 (ESP-12E Module)`
   - ESP32-CAM → `Tools → Board → AI Thinker ESP32-CAM`

> 💡 **国内下载慢**：开发板包动辄几百 MB，建议用 [Arduino 中文社区镜像](https://www.arduino.cn/thread-5838-1-1.html) 或直接下离线包。

---

# 二、ESP8266 NodeMCU 入门

## 2.1 引脚速查

NodeMCU V3 的丝印和实际 GPIO 编号**对不上**，这是新手第一个坑：

| 丝印 | 实际 GPIO | 备注 |
|:--|:--|:--|
| D0 | GPIO16 | 只能做普通 GPIO / 深度睡眠唤醒 |
| D1 | GPIO5 | ✅ 常用，支持 I2C SCL |
| D2 | GPIO4 | ✅ 常用，支持 I2C SDA |
| D3 | GPIO0 | ⚠️ 拉低进烧录模式，慎用 |
| D4 | GPIO2 | ⚠️ 板载 LED（低电平点亮） |
| D5 | GPIO14 | ✅ 支持 SPI |
| D6 | GPIO12 | ✅ 支持 SPI |
| D7 | GPIO13 | ✅ 支持 SPI |
| D8 | GPIO15 | ⚠️ 启动必须为低，接了外设要小心 |
| A0 | ADC0 | 模拟输入，**输入范围 0~1.0V** |

**代码里永远用 GPIO 编号**，不用 D 编号：

```cpp
pinMode(5, OUTPUT);      // 对：D1 = GPIO5
// pinMode(D1, OUTPUT);  // 错：Arduino 的 ESP8266 core 不认 D 开头
```

**供电注意**：NodeMCU 的 3.3V 输出**最多给 200mA 左右**。接继电器、舵机、电机必须**外接电源**，否则会重启或烧稳压芯片。

## 2.2 第一个程序：点灯

板载 LED 接在 **GPIO2（D4）**，且是**低电平点亮**（反逻辑）：

```cpp
void setup() {
  pinMode(2, OUTPUT);
  Serial.begin(115200);
  Serial.println("Hello ESP8266");
}

void loop() {
  digitalWrite(2, LOW);   // 低电平 = 亮
  delay(500);
  digitalWrite(2, HIGH);  // 高电平 = 灭
  delay(500);
}
```

**烧录设置**（Tools 菜单）：

| 项 | 值 |
|:--|:--|
| Board | NodeMCU 1.0 (ESP-12E Module) |
| Upload Speed | 115200（不稳就降到 9600） |
| Flash Size | 4MB (FS:2MB OTA:~1019KB) |
| Port | 你的 COM 口 / `/dev/ttyUSB0` |

> 如果报 `Failed to connect to ESP8266`：拔掉再插、换根线（**很多 USB 线只有供电没有数据线**）、把 Upload Speed 降到 9600。

## 2.3 GPIO / PWM / ADC

### PWM 调光

```cpp
void setup() {
  pinMode(2, OUTPUT);
  analogWriteRange(1023);   // ESP8266 默认 0~1023
  analogWriteFreq(1000);    // 1kHz，呼吸灯太闪就调高
}

void loop() {
  for (int i = 0; i <= 1023; i += 5) {
    analogWrite(2, i);
    delay(3);
  }
  for (int i = 1023; i >= 0; i -= 5) {
    analogWrite(2, i);
    delay(3);
  }
}
```

> ⚠️ ESP8266 的 `analogWrite` 是软件模拟 PWM，频率高时占 CPU。要精准 PWM 用 `ESP8266PWM` 库。

### ADC 读电压

```cpp
void setup() { Serial.begin(115200); }

void loop() {
  int raw = analogRead(A0);           // 0~1023
  float volt = raw / 1023.0 * 1.0;    // 满量程 1.0V
  float real = volt * 3.2;            // 分压比：假设 220k + 100k
  Serial.printf("raw=%d  vol=%.3fV  real=%.2fV\n", raw, volt, real);
  delay(1000);
}
```

**关键**：A0 输入**最大 1.0V**！测电池电压必须分压。 10k + 10k 分压 → 量程 2V； 220k + 100k → 量程约 3.2V。

## 2.4 WiFi 联网：STA 模式

```cpp
#include <ESP8266WiFi.h>

const char* SSID = "你的WiFi名";
const char* PASS = "你的密码";

void setup() {
  Serial.begin(115200);
  WiFi.mode(WIFI_STA);
  WiFi.begin(SSID, PASS);

  Serial.print("连接中");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println();
  Serial.print("已连接，IP: ");
  Serial.println(WiFi.localIP());
}

void loop() {}
```

> ⚠️ **ESP8266 只支持 2.4GHz WiFi**，连不上 5G 路由器 —— 这是最高频的「连不上」原因。

### AP 模式（板子自己开热点）

```cpp
#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>

ESP8266WebServer server(80);

void setup() {
  WiFi.mode(WIFI_AP);
  WiFi.softAP("ESP8266-LED", "12345678");
  Serial.println(WiFi.softAPIP());   // 默认 192.168.4.1

  server.on("/", []() {
    server.send(200, "text/html; charset=utf-8",
      "<h1>控制面板</h1>"
      "<a href='/on'>开灯</a> | <a href='/off'>关灯</a>");
  });

  server.on("/on", []() {
    digitalWrite(2, LOW);
    server.send(200, "text/plain", "LED ON");
  });

  server.on("/off", []() {
    digitalWrite(2, HIGH);
    server.send(200, "text/plain", "LED OFF");
  });

  pinMode(2, OUTPUT);
  server.begin();
}

void loop() { server.handleClient(); }
```

烧完后手机连 `ESP8266-LED`，浏览器打开 `192.168.4.1`，就能点灯了。**这就是最小可用的「物联网」**。 

## 2.5 综合小项目：Web 控制灯 + 温度上报

```cpp
#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>
#include <DHT.h>

#define DHTPIN 4          // GPIO4 = D2
#define DHTTYPE DHT11

ESP8266WebServer server(80);
DHT dht(DHTPIN, DHTTYPE);

void setup() {
  Serial.begin(115200);
  pinMode(2, OUTPUT);
  digitalWrite(2, HIGH);

  dht.begin();

  WiFi.mode(WIFI_STA);
  WiFi.begin("SSID", "PASS");
  while (WiFi.status() != WL_CONNECTED) delay(500);
  Serial.println(WiFi.localIP());

  // CORS 头，方便网页/JS 调用
  server.enableCORS(true);

  server.on("/led", []() {
    String s = server.arg("s");
    digitalWrite(2, s == "on" ? LOW : HIGH);
    server.send(200, "application/json", "{\"led\":\"" + s + "\"}");
  });

  server.on("/temp", []() {
    float t = dht.readTemperature();
    float h = dht.readHumidity();
    if (isnan(t) || isnan(h)) {
      server.send(500, "application/json", "{\"error\":\"read fail\"}");
      return;
    }
    server.send(200, "application/json",
      "{\"temp\":" + String(t, 1) + ",\"hum\":" + String(h, 1) + "}");
  });

  server.begin();
}

void loop() { server.handleClient(); }
```

访问 `http://<IP>/led?s=on` 和 `http://<IP>/temp` 即可。

---

# 三、ESP32-CAM 进阶

## 3.1 与 ESP8266 的关键差异

- **双核 240MHz**：一个核跑 WiFi 协议栈，一个核跑你的代码
- **PSRAM**：做 JPEG 缓冲必需，没它只能跑 QVGA 分辨率
- **摄像头接口**：8 位并行 DVP，直连 OV2640
- **没有板载 USB 串口**：必须用外部 USB-TTL 模块

## 3.2 摄像头引脚映射（AI Thinker 板）

| 功能 | GPIO | | 功能 | GPIO |
|:--|:--|--|:--|:--|
| PWDN | 32 | | D0 (Y2) | 5 |
| RESET | -1 (未接) | | D1 (Y3) | 18 |
| XCLK | 0 | | D2 (Y4) | 19 |
| SIOD (SDA) | 26 | | D3 (Y5) | 21 |
| SIOC (SCL) | 27 | | D4 (Y6) | 36 |
| D7 (Y9) | 16 | | D5 (Y7) | 39 |
| D6 (Y8) | 17 | | D6 (Y8) | 34 |
| VSYNC | 25 | | D7 (Y9) | 35 |
| HREF | 23 | | PCLK | 22 |
| 闪光灯 | 4 | | SD 卡 | 2/14/15 |

**可用 GPIO 只剩：12、13、14、15、16**（16 还被 SD 占用）。这就是为什么我说 ESP32-CAM 不适合当通用控制板。

## 3.3 接线烧录（关键步骤）

ESP32-CAM 没有 USB，需要 USB-TTL（CH340 / CP2102 都行）：

```
USB-TTL          ESP32-CAM
 GND  ───────────  GND        ← 必须共地
 5V   ───────────  5V         ← 用 5V，3.3V 供电不足
 TX   ───────────  U0R (RX)
 RX   ───────────  U0T (TX)
```

**烧录流程**（没做好会一直 `Failed to connect`）：

1. **IO0 接 GND**（进入下载模式）
2. 按一下 **RST** 键（或断开再插 5V）
3. Arduino IDE 点上传
4. 看到 `Connecting...` 转圈后出现 `Writing at 0x0000...` 就是通了
5. 烧完**拔掉 IO0 的 GND**，再按一次 RST

> ⚠️ 供电是最大坑：ESP32-CAM 启动瞬间电流可到 300mA+，**USB-TTL 的 3.3V 针脚给不起**。务必接 5V 针（板上有 LDO）。

## 3.4 视频流服务器（官方例程）

`文件 → 示例 → ESP32 → Camera → CameraWebServer`

改两处：

```cpp
// 1) 选对摄像头型号
#define CAMERA_MODEL_AI_THINKER   // 取消注释这行

// 2) 填 WiFi
const char* ssid = "你的WiFi";
const char *password = "你的密码";
```

烧录后串口会打印：

```
Camera Ready! Use 'http://192.168.1.123' to connect
```

浏览器打开这个地址，就能看到实时画面面板，可调分辨率、亮度、特效。

**核心 API**（自己写时用）：

```cpp
#include "esp_camera.h"
#include "camera_pins.h"

void setup() {
  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer   = LEDC_TIMER_0;
  config.pin_d0       = Y2_GPIO_NUM;
  config.pin_d1       = Y3_GPIO_NUM;
  // ... 完整引脚见 camera_pins.h
  config.pin_pclk     = PCLK_GPIO_NUM;
  config.pin_vsync    = VSYNC_GPIO_NUM;
  config.pin_href     = HREF_GPIO_NUM;
  config.pin_sscb_sda = SIOD_GPIO_NUM;
  config.pin_sscb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn     = PWDN_GPIO_NUM;
  config.pin_reset    = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;

  // PSRAM 决定能跑多大分辨率
  if (psramFound()) {
    config.frame_size   = FRAMESIZE_UXGA;   // 1600x1200
    config.jpeg_quality = 10;
    config.fb_count     = 2;
  } else {
    config.frame_size   = FRAMESIZE_SVGA;   // 800x600
    config.jpeg_quality = 12;
    config.fb_count     = 1;
  }

  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Camera init failed: 0x%x\n", err);
    return;
  }
}
```

**分辨率与帧率对照**（AI Thinker + PSRAM）：

| 分辨率 | 尺寸 | 可用帧率 | 说明 |
|:--|:--|:--|:--|
| QVGA | 320×240 | ~25fps | 流畅，适合图传 |
| VGA | 640×480 | ~15fps | 平衡 |
| SVGA | 800×600 | ~10fps | |
| UXGA | 1600×1200 | ~5fps | 只适合拍照 |

## 3.5 拍照存 SD 卡

```cpp
#include "esp_camera.h"
#include "FS.h"
#include "SD_MMC.h"

void takePhoto() {
  camera_fb_t *fb = esp_camera_fb_get();
  if (!fb) {
    Serial.println("拍照失败");
    return;
  }

  String path = "/photo_" + String(millis()) + ".jpg";
  fs::FS &fs = SD_MMC;
  File file = fs.open(path.c_str(), FILE_WRITE);
  if (file) {
    file.write(fb->buf, fb->len);
    Serial.printf("已保存 %s (%u bytes)\n", path.c_str(), fb->len);
  }
  file.close();
  esp_camera_fb_return(fb);   // ⚠️ 必须归还，否则内存泄漏
}

void setup() {
  Serial.begin(115200);
  // camera 初始化同上
  if (!SD_MMC.begin("/sdcard", true)) {   // true = 1-bit 模式
    Serial.println("SD 卡挂载失败");
  }
}
```

> ⚠️ `esp_camera_fb_return(fb)` 忘了调用 → 几次之后就 `fb_get` 返回 NULL，看起来像「摄像头坏了」。

---

# 四、从两块板子到一台机器人

## 4.1 典型双板架构

```
       ┌─────────────────┐          ┌──────────────────┐
       │   ESP32-CAM     │          │   ESP8266        │
       │   （眼睛）       │          │   （手脚）        │
       │                 │          │                  │
       │  OV2640 摄像头   │          │  L298N 电机驱动   │
       │  MJPEG 图传      │  MQTT    │  舵机云台         │
       │  麦克风(可选)     │ ◄──────► │  LED / 蜂鸣器     │
       └─────────────────┘          └──────────────────┘
                │                            │
                └──────────┬─────────────────┘
                           ▼
                    ┌──────────────┐
                    │  MQTT Broker │
                    │ (路由器/树莓派)│
                    └──────────────┘
                           ▲
                           │
                    ┌──────────────┐
                    │  手机/电脑    │
                    │  （遥控端）    │
                    └──────────────┘
```

**为什么这么分工**：
- ESP32-CAM 的 GPIO 被摄像头占完，驱动不了电机
- ESP8266 引脚富裕，但算力和内存跑不动视频
- 两块各司其职，靠 MQTT 解耦

## 4.2 通信方案对比

| 方案 | 优点 | 缺点 | 适用 |
|:--|:--|:--|:--|
| **HTTP 轮询** | 最简单，无需服务器 | 延迟高、费流量 | 单次查询（温度） |
| **MQTT** | 低延迟、双向、省电 | 需要 Broker | **遥控指令** ✅ |
| **ESP-NOW** | 无路由器直连、延迟极低 | 只支持 ESP 间、距离短 | 板间高速通信 |
| **WebSocket** | 浏览器直连 | 实现复杂 | 实时图传控制 |

**推荐组合**：图传用 HTTP（MJPEG），控制指令用 MQTT。

### MQTT 快速上手

需要一个 Broker，最省事的是：

```bash
# Docker 一条命令起一个
docker run -d --name mosquitto -p 1883:1883 \
  -p 9001:9001 eclipse-mosquitto
```

或直接用公共测试服务器：`test.mosquitto.org:1883`（不安全，仅测试）。

**发布端（手机 / 电脑）**：

```bash
# 命令行测试
mosquitto_pub -h test.mosquitto.org -t "car/cmd" -m "forward"
mosquitto_pub -h test.mosquitto.org -t "car/cmd" -m "stop"
```

**订阅端（ESP8266 控制电机）**：

```cpp
#include <ESP8266WiFi.h>
#include <PubSubClient.h>

const char* MQTT_SERVER = "test.mosquitto.org";
const int   MQTT_PORT   = 1883;
const char* TOPIC_CMD   = "car/cmd";

// L298N 接法：IN1/IN2 控左轮方向，IN3/IN4 控右轮
#define IN1 5    // D1
#define IN2 4    // D2
#define IN3 14   // D5
#define IN4 12   // D6

WiFiClient espClient;
PubSubClient client(espClient);

void forward() { digitalWrite(IN1,1); digitalWrite(IN2,0);
                 digitalWrite(IN3,1); digitalWrite(IN4,0); }
void backward() { digitalWrite(IN1,0); digitalWrite(IN2,1);
                  digitalWrite(IN3,0); digitalWrite(IN4,1); }
void turnLeft() { digitalWrite(IN1,0); digitalWrite(IN2,1);
                  digitalWrite(IN3,1); digitalWrite(IN4,0); }
void turnRight() { digitalWrite(IN1,1); digitalWrite(IN2,0);
                   digitalWrite(IN3,0); digitalWrite(IN4,1); }
void stop() { digitalWrite(IN1,0); digitalWrite(IN2,0);
              digitalWrite(IN3,0); digitalWrite(IN4,0); }

void callback(char* topic, byte* payload, unsigned int len) {
  String cmd = "";
  for (unsigned i = 0; i < len; i++) cmd += (char)payload[i];
  Serial.println("收到: " + cmd);

  if (cmd == "forward")      forward();
  else if (cmd == "backward") backward();
  else if (cmd == "left")     turnLeft();
  else if (cmd == "right")    turnRight();
  else if (cmd == "stop")     stop();
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("MQTT 连接中...");
    if (client.connect("ESP8266-Car")) {
      Serial.println("ok");
      client.subscribe(TOPIC_CMD);
    } else {
      Serial.printf("失败 rc=%d\n", client.state());
      delay(2000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);
  stop();

  WiFi.begin("SSID", "PASS");
  while (WiFi.status() != WL_CONNECTED) delay(500);

  client.setServer(MQTT_SERVER, MQTT_PORT);
  client.setCallback(callback);
}

void loop() {
  if (!client.connected()) reconnect();
  client.loop();
}
```

## 4.3 硬件构成清单

做一台完整小车，除两块板子外还需要：

| 部件 | 规格 | 价格 | 说明 |
|:--|:--|:--|:--|
| 小车底盘 | 2WD 或 4WD + 轮子 | 20~40 | 亚克力/铝合金 |
| 电机驱动 | **L298N** 或 TB6612 | 8~15 | L298N 发热大，TB6612 更好 |
| 电池 | 18650 ×2 + 电池盒 | 20 | **必须独立供电**，别从板子取 |
| 降压模块 | LM2596 → 5V | 5 | 电池 7.4V 降 5V 给板子 |
| 杜邦线 | 母对母 / 公对母 | 5 | 多备点 |

**供电拓扑**（这一步错了必烧板）：

```
18650 ×2 (7.4V)
     │
     ├──► L298N 电机供电（直连，电机要大电流）
     │
     └──► LM2596 降压 → 5V ──┬──► ESP8266 VIN
                              └──► ESP32-CAM 5V
     └──► 所有 GND 必须共地！
```

## 4.4 进阶方向

学完小车之后，可以往这几个方向发展：

| 方向 | 关键技术 | 参考 |
|:--|:--|:--|
| **接入 ROS2** | micro-ROS（ESP32 支持） | `industry/02-机器人行业专题` |
| **图像识别** | 改用 ESP32-S3 + 边缘推理（TFLite Micro） | — |
| **多机协同** | ESP-NOW 组网 | — |
| **OTA 升级** | ArduinoOTA | — |
| **低功耗** | 深度睡眠 + 定时唤醒 | — |

---

# 五、排障手册（按症状查）

| 症状 | 可能原因 | 排查 |
|:--|:--|:--|
| `Failed to connect to ESP8266` | USB 线无数据线 / 驱动没装 / 波特率太高 | 换线 → 看设备管理器 → 降到 9600 |
| `Permission denied: /dev/ttyUSB0` | Linux 权限 | `sudo usermod -aG dialout $USER` 后重新登录 |
| 烧录成功但无输出 | 波特率不对 / 板子没复位 | 串口监视器改 115200，按 RST |
| WiFi 连不上 | **连的 5G** / 密码错 / 信号弱 | 换 2.4G；`WiFi.printDiag(Serial)` 看状态码 |
| 8266 一直重启 | 供电不足（接了外设） | 外接电源，别用板载 3.3V |
| ESP32-CAM 烧录失败 | IO0 没接地 / 供电不足 | IO0→GND，按 RST，用 5V 供电 |
| ESP32-CAM 图像全黑 | 摄像头排线没插紧 / 供电不足 | 重新插 FPC 排线，**断电操作** |
| `Camera init failed: 0x105` | PSRAM 未启用 / 引脚配置错 | Tools → PSRAM → Enabled |
| 拍几张后 `fb_get` 返回 NULL | 忘了 `esp_camera_fb_return` | 每次 `fb_get` 必配对 `fb_return` |
| SD 卡挂载失败 | 卡格式不对 / 容量过大 | 格成 **FAT32**，≤32GB |
| MQTT 一直重连 | 服务器地址错 / 网络不通 | `ping test.mosquitto.org` |
| 电机不转 / 转一下就停 | 电池电压不够 / L298N 过热 | 量电压；加散热片或换 TB6612 |
| 小车动但摄像头卡 | WiFi 带宽被占满 | 降分辨率到 VGA，降帧率 |

---

# 六、坑位汇总

**硬件层**
1. USB 线有「只能充电」的，烧录一定用数据线
2. ESP32-CAM 的 3.3V 针脚电流不够，**必须接 5V**
3. 电机和板子**必须共地**，且电机单独供电
4. 摄像头 FPC 排线脆弱，**插拔必须断电**
5. 8266 的 A0 只能输入 1.0V，测高压必须分压

**软件层**
6. 8266 代码用 GPIO 编号，不用 D 编号
7. `esp_camera_fb_get` 必须配 `esp_camera_fb_return`
8. `String` 拼接在内存小的板子上容易碎片化，长时间运行建议用小缓冲区
9. 8266 的 `analogWrite` 是软件 PWM，占 CPU
10. 烧 ESP32-CAM 忘了拔 IO0 的接地线 → 程序跑不起来

**网络层**
11. ESP 系全系**只支持 2.4GHz**，5G 连不上
12. 公共 MQTT 服务器（test.mosquitto.org）**不安全**，正式用请自建
13. 视频流 + 高频控制指令会抢带宽，建议双通道分离

---

# 七、学习路线

```
第 1 周：ESP8266 打基础
  ├── 点灯 / 串口打印
  ├── PWM 呼吸灯
  ├── ADC 读电压（加分压）
  └── WiFi 联网 + Web 服务器

第 2 周：ESP32-CAM 拿视觉
  ├── 接线烧录跑通 CameraWebServer
  ├── 调分辨率 / 亮度 / 特效
  └── 拍照存 SD 卡

第 3-4 周：小车实战
  ├── 装底盘 + 电机驱动
  ├── MQTT 遥控单板跑通
  ├── 双板联调（图传 + 控制）
  └── 调 PID / 加避障传感器

第 5 周+：进阶
  ├── micro-ROS 接入
  └── 边缘 AI 推理
```

**参考资料**
- ESP8266 Arduino Core 文档：https://arduino-esp8266.readthedocs.io/
- ESP32 Arduino Core 文档：https://docs.espressif.com/projects/arduino-esp32/
- Random Nerd Tutorials（例程最多）：https://randomnerdtutorials.com/
- OV2640 数据手册：OmniVision 官网
- L298N 数据手册：ST 官网

---

## 相关文档

- `01-C-C++嵌入式构建流程.md` — 从源码到固件的构建链路
- `../../industry/02-机器人行业专题.md` — 机器人工位的能力地图
- `../devops/practices/08-AI-Agent辅助CI排障实战.md` — AI 辅助排障的另一条思路
