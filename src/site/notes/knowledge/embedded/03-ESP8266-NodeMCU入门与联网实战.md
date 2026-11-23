---
{"dg-publish":true,"permalink":"/knowledge/embedded/03-ESP8266-NodeMCU入门与联网实战/","title":"ESP8266 NodeMCU 入门与联网实战","tags":["ESP8266","NodeMCU","Arduino","ESP8266_RTOS_SDK","FreeRTOS","idf.py","GPIO","PWM","LEDC","ADC","WiFi","WebServer","OLED","SSD1306","U8g2","天气","嵌入式","物联网"],"dg-note-properties":{"title":"ESP8266 NodeMCU 入门与联网实战","date":"2026-11-14","tags":["ESP8266","NodeMCU","Arduino","ESP8266_RTOS_SDK","FreeRTOS","idf.py","GPIO","PWM","LEDC","ADC","WiFi","WebServer","OLED","SSD1306","U8g2","天气","嵌入式","物联网"]}}
---


# 🔌 ESP8266 NodeMCU 入门与联网实战

> 一块 10 元的小板子，从插线点灯一路做到「联网抓天气 + OLED 中文显示」的桌面摆件。
> ESP8266 的价值在于**便宜、引脚富裕、插线即烧**：先打牢 GPIO / WiFi / 传感器基础，再去做联网应用，是最不容易劝退的路径。

---

## 📑 本页导航

| 章节 | 内容 | 难度 |
|:--|:--|:--|
| 一、认识 NodeMCU | 芯片规格、D 编号陷阱、CH340 驱动 | ⭐ |
| 二、开发环境 | 板卡包、烧录设置、开发方式对比 | ⭐ |
| 三、点灯起步 | GPIO 输出、板载 LED 反逻辑 | ⭐ |
| 四、PWM 与 ADC | 呼吸灯、读电压与分压 | ⭐⭐ |
| 五、WiFi 联网 | STA、AP、Web 服务器、JSON 接口 | ⭐⭐ |
| 六、接 OLED 显示 | I2C 接线、U8g2、中文字体、图标 | ⭐⭐⭐ |
| 七、天气助手实战 | 心知天气 + 解析 + 显示 + 稳定性加固 | ⭐⭐⭐⭐ |
| 八、脱开 Arduino | ESP8266_RTOS_SDK：工具链、构建、WiFi、外设 | ⭐⭐⭐⭐ |
| 九、排障手册 | 按症状查 | — |
| 十、坑位汇总 | 一次性避坑 | — |

---

# 一、认识 NodeMCU

## 1.1 关键规格

| 项 | 值 |
|:--|:--|
| 芯片 | ESP8266EX（单核 Tensilica L106 @ 80MHz，可超到 160MHz）|
| RAM | ~80KB 可用（堆约 40KB，**很紧张**）|
| Flash | 4MB（NodeMCU V3 常见）|
| WiFi | 802.11 b/g/n，**仅 2.4GHz** |
| GPIO | ~11 个可用 |
| 板载 USB | ✅ CH340，插线即烧 |
| ADC | 1 路，**输入范围 0~1.0V** |
| 价格 | ~10 元 |

**为什么从 8266 起步**：插上 USB 就能烧，不用折腾接线；引脚没被外设吃光，GPIO 实验随便做。出错概率低，适合建立信心。

## 1.2 引脚速查：D 编号是最大的坑

NodeMCU V3 的丝印 `D1/D2` 和实际 GPIO 编号**对不上**：

| 丝印 | 实际 GPIO | 备注 |
|:--|:--|:--|
| D0 | GPIO16 | 只能做普通 GPIO / 深度睡眠唤醒 |
| D1 | GPIO5 | ✅ 常用，支持 I2C SCL |
| D2 | GPIO4 | ✅ 常用，支持 I2C SDA |
| D3 | GPIO0 | ⚠️ 拉低进烧录模式，慎用 |
| D4 | GPIO2 | ⚠️ 板载 LED（**低电平点亮**）|
| D5 | GPIO14 | ✅ 支持 SPI |
| D6 | GPIO12 | ✅ 支持 SPI |
| D7 | GPIO13 | ✅ 支持 SPI |
| D8 | GPIO15 | ⚠️ 启动必须为低，接外设要小心 |
| A0 | ADC0 | 模拟输入，**0~1.0V** |

**代码里永远用 GPIO 编号**：

```cpp
pinMode(5, OUTPUT);      // 对：D1 = GPIO5
// pinMode(D1, OUTPUT);  // 错：ESP8266 core 不认 D 开头宏
```

**供电红线**：板载 3.3V 输出**最多约 200mA**。接继电器、舵机、电机必须外接电源，否则重启或烧稳压芯片。

## 1.3 CH340 驱动

NodeMCU V3 用的是 **CH340**（不是 CP2102）。Windows 10/11 可能不认。

```bash
# Windows：装 ch341ser.exe → 设备管理器出现 COMx
# macOS：WCH 官网下 CH34xVCPDriver.pkg → /dev/tty.wchusbserial1420
# Linux：内核自带
dmesg | tail -20      # 看到 ch341-uart converter now attached to ttyUSB0
ls /dev/ttyUSB*
sudo usermod -aG dialout $USER   # 避免每次 sudo
```

> ⚠️ Linux 不加 `dialout` 组会报 `Permission denied: '/dev/ttyUSB0'`。

---

# 二、开发环境

## 2.1 安装 Arduino IDE

1. 装 **Arduino IDE 2.x**
2. `文件 → 首选项 → 附加开发板管理器网址` 填：
   ```
   http://arduino.esp8266.com/stable/package_esp8266com_index.json
   ```
3. 开发板管理器搜 `esp8266` → 安装 `esp8266 by ESP8266 Community`
4. 选板子：`Tools → Board → NodeMCU 1.0 (ESP-12E Module)`

> 💡 开发板包几百 MB，国内下载慢可用 Arduino 中文社区镜像或离线包。

## 2.2 烧录设置

| 项 | 值 |
|:--|:--|
| Board | NodeMCU 1.0 (ESP-12E Module) |
| Upload Speed | 921600（不稳降 115200，再不行 9600）|
| CPU Frequency | 80 MHz |
| Flash Size | 4MB (FS:2MB OTA:~1019KB) |
| Erase Flash | Only Sketch |

## 2.3 三套开发方式

| 方式 | 语言 | 上手 | 适合 |
|:--|:--|:--|:--|
| **Arduino IDE / PlatformIO** | C++ | ⭐⭐ | **本文默认**，例子最多 |
| MicroPython | Python | ⭐ | 快速原型 |
| **ESP8266_RTOS_SDK** | C | ⭐⭐⭐⭐ | 产品级精细控制，见第八章 |
| ESP8266_NONOS_SDK | C | ⭐⭐⭐⭐ | 无 RTOS，最贴硬件 |

---

# 三、点灯起步

板载 LED 在 **GPIO2（D4）**，**低电平点亮**（反逻辑）：

```cpp
void setup() {
  pinMode(2, OUTPUT);
  Serial.begin(115200);
  Serial.println("Hello ESP8266");
}

void loop() {
  digitalWrite(2, LOW);   // 低 = 亮
  delay(500);
  digitalWrite(2, HIGH);  // 高 = 灭
  delay(500);
}
```

> 报 `Failed to connect to ESP8266`：换 USB 线（**很多线只有供电没有数据线**）、降波特率、拔插一次。

---

# 四、PWM 与 ADC

## 4.1 PWM 呼吸灯

```cpp
void setup() {
  pinMode(2, OUTPUT);
  analogWriteRange(1023);   // 默认 0~1023
  analogWriteFreq(1000);    // 1kHz，太闪就调高
}

void loop() {
  for (int i = 0; i <= 1023; i += 5) { analogWrite(2, i); delay(3); }
  for (int i = 1023; i >= 0; i -= 5) { analogWrite(2, i); delay(3); }
}
```

> ⚠️ 8266 的 `analogWrite` 是**软件模拟 PWM**，频率高时吃 CPU。要精准多路 PWM 用 `ESP8266_NewPing` 之外的分时库，或直接换 ESP32。

## 4.2 ADC 读电压

```cpp
void loop() {
  int raw = analogRead(A0);           // 0~1023
  float volt = raw / 1023.0 * 1.0;    // 满量程 1.0V
  float real = volt * 3.2;            // 分压比：220k + 100k
  Serial.printf("raw=%d  vol=%.3fV  real=%.2fV\n", raw, volt, real);
  delay(1000);
}
```

**A0 最大只能 1.0V**！测电池必须分压：

| 分压电阻 | 测量量程 |
|:--|:--|
| 10k + 10k | 2.0V |
| 100k + 47k | ~3.1V |
| 220k + 100k | ~3.2V |

---

# 五、WiFi 联网

## 5.1 STA 模式（连路由器）

```cpp
#include <ESP8266WiFi.h>

const char* SSID = "你的WiFi名";
const char* PASS = "你的密码";

void setup() {
  Serial.begin(115200);
  WiFi.mode(WIFI_STA);
  WiFi.begin(SSID, PASS);

  Serial.print("连接中");
  while (WiFi.status() != WL_CONNECTED) { delay(500); Serial.print("."); }
  Serial.println();
  Serial.print("已连接，IP: ");
  Serial.println(WiFi.localIP());
}

void loop() {}
```

> ⚠️ **只支持 2.4GHz**，连不上 5G 路由器 —— 最高频的「连不上」原因。

## 5.2 AP 模式（自己开热点）

```cpp
#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>

ESP8266WebServer server(80);

void setup() {
  WiFi.mode(WIFI_AP);
  WiFi.softAP("ESP8266-LED", "12345678");
  Serial.println(WiFi.softAPIP());   // 默认 192.168.4.1

  pinMode(2, OUTPUT);

  server.on("/", []() {
    server.send(200, "text/html; charset=utf-8",
      "<h1>控制面板</h1><a href='/on'>开灯</a> | <a href='/off'>关灯</a>");
  });
  server.on("/on",  []() { digitalWrite(2, LOW);  server.send(200, "text/plain", "LED ON");  });
  server.on("/off", []() { digitalWrite(2, HIGH); server.send(200, "text/plain", "LED OFF"); });

  server.begin();
}

void loop() { server.handleClient(); }
```

手机连 `ESP8266-LED`，浏览器开 `192.168.4.1` 就能点灯 —— **这就是最小可用的物联网**。

## 5.3 Web 控灯 + 测温 API

```cpp
#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>
#include <DHT.h>

#define DHTPIN  4          // GPIO4 = D2
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

  server.enableCORS(true);   // 方便网页/JS 直接调用

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

- `http://<IP>/led?s=on` / `?s=off` 控灯
- `http://<IP>/temp` 返回 `{"temp":26.3,"hum":48.5}`

---

# 六、接 OLED 显示

## 6.1 为什么选 SSD1306

| 对比 | SSD1306（0.96" OLED）| ST7735（彩屏）|
|:--|:--|:--|
| 接线 | 4 根（I2C）| 8 根（SPI）|
| 功耗 | 低，无背光 | 高，有背光 |
| 库支持 | U8g2 完善 | TFT_eSPI |
| 价格 | ~10 元 | ~20 元 |

第一块屏就选 SSD1306，接线少、不易出错。

## 6.2 I2C 接线

| OLED 引脚 | NodeMCU 引脚 | GPIO |
|:--|:--|:--|
| VCC | 3V3 | — |
| GND | GND | — |
| SCL | D1 | GPIO5 |
| SDA | D2 | GPIO4 |

**先确认地址**（`0x3C` 或 `0x3D`）：

```cpp
#include <Wire.h>

void setup() {
  Serial.begin(115200);
  Wire.begin(D2, D1);
  for (uint8_t addr = 1; addr < 127; addr++) {
    Wire.beginTransmission(addr);
    if (Wire.endTransmission() == 0) {
      Serial.print("Found: 0x"); Serial.println(addr, HEX);
    }
  }
}

void loop() {}
```

没扫到？查：杜邦线虚接 → 供电 → SDA/SCL 接反 → 模块坏。

## 6.3 库与对象

装 **U8g2**（库管理器搜 `U8g2`），然后：

```cpp
#include <U8g2lib.h>

// 全缓冲(F) + 硬件 I2C，显式指定 SCL=D1 / SDA=D2
U8G2_SSD1306_128X64_NONAME_F_HW_I2C u8g2(
    U8G2_R0, /* reset=*/ U8X8_PIN_NONE,
    /* clock=*/ D1, /* data=*/ D2);

void setup() {
  u8g2.begin();
  u8g2.enableUTF8Print();        // 允许 print() 输出中文
  u8g2.setFont(u8g2_font_wqy12_t_gb2312);
  u8g2.clearBuffer();
  u8g2.drawUTF8(0, 14, "中文显示测试");
  u8g2.sendBuffer();
}

void loop() {}
```

要点：
- **必须用 `_F_`（全缓冲）**才能用 `drawUTF8` 的完整字形
- `U8G2_R0` 是 0°；屏装反了改 `U8G2_R2`

## 6.4 中文字体选型

| 字体 | 字号 | 字符集 | Flash 占用 | 适用 |
|:--|:--|:--|:--|:--|
| `u8g2_font_wqy12_t_gb2312` | 12px | GB2312 常用字 | ~80KB | 正文 |
| `u8g2_font_wqy16_t_gb2312` | 16px | GB2312 | ~130KB | 标题/主信息 |
| `u8g2_font_wqy12_t_chinese1` | 12px | 少量汉字 | ~30KB | Flash 紧张时 |

> 4MB Flash 装个 GB2312 全集毫无压力，主信息直接上 wqy16 更清晰。

## 6.5 8×8 小图标（字模）

不引图片资源，用 `drawXBMP()`：

```cpp
static const uint8_t ICON_SUN_8[]   PROGMEM = {0x18,0x18,0x7E,0x3C,0x3C,0x7E,0x18,0x18};
static const uint8_t ICON_CLOUD_8[] PROGMEM = {0x00,0x38,0x7C,0xFE,0xFE,0x7C,0x00,0x00};
static const uint8_t ICON_RAIN_8[]  PROGMEM = {0x38,0x7C,0xFE,0x00,0x24,0x24,0x48,0x48};

void drawWeatherIcon(int x, int y, const char* code) {
  int c = atoi(code);
  const uint8_t* bmp = ICON_CLOUD_8;
  if (c == 0)                  bmp = ICON_SUN_8;
  else if (c >= 10 && c <= 25) bmp = ICON_RAIN_8;
  u8g2.drawXBMP(x, y, 8, 8, bmp);
}
```

## 6.6 布局坐标速查（128×64）

```
y=0   ─────────────────────
y=12  │ 城市        │ 10:40  │   ← 12px 中文 + 6x10
y=15  ├───────────────────────┤   ← drawHLine
y=28  │ [图标] 天气文本        │   ← 16px
y=42  │              26C      │   ← logisoso24
y=60  │ OK                   │   ← 6x10
y=64  ─────────────────────
```

**右对齐的正确做法**：先 `getStrWidth()` 量宽，再用 `128 - w` 定位，别手调坐标。

```cpp
u8g2.setFont(u8g2_font_logisoso24_tn);
String t = String(wx.temp) + "C";
int w = u8g2.getStrWidth(t.c_str());
u8g2.drawStr(126 - w, 42, t.c_str());
```

---

# 七、天气助手实战

把「联网取数据」和「屏幕显示」合起来，做一个开机自动连 WiFi、每 10 分钟刷新的桌面天气钟。

## 7.1 数据源：心知天气

1. 注册 `seniverse.com` → 控制台
2. 产品管理 → 添加产品 → **天气数据 API（免费版）**
3. 基本信息 → 复制「**私钥**」

免费版每天 400 次调用，实况接口够用（10 分钟一次 = 144 次/天）。

```
https://api.seniverse.com/v3/weather/now.json?key=<私钥>&location=hangzhou&language=zh-Hans&unit=c
```

返回只用这几段：

```json
{
  "results": [{
    "location": { "name": "杭州" },
    "now": { "text": "多云", "code": "4", "temperature": "26" },
    "last_update": "2026-11-14T10:40:00+08:00"
  }]
}
```

> 💡 **不想注册**？用 Open-Meteo（免注册无 key）：
> `https://api.open-meteo.com/v1/forecast?latitude=30.28&longitude=120.15&current=temperature_2m,weather_code,relative_humidity_2m&timezone=Asia%2FShanghai`

## 7.2 依赖与数据结构

```cpp
#include <ESP8266WiFi.h>
#include <ArduinoJson.h>      // v6.x
#include <Wire.h>
#include <U8g2lib.h>

const char* WIFI_SSID = "your-ssid";
const char* WIFI_PASS = "your-password";
const char* API_KEY   = "your-seniverse-key";
const char* CITY      = "hangzhou";
const char* HOST      = "api.seniverse.com";

const uint32_t FETCH_INTERVAL_MS = 10UL * 60UL * 1000UL;
const uint16_t HTTP_TIMEOUT_MS   = 6000;

struct Weather {
  char city[16]    = "----";
  char text[24]    = "等待数据";
  char temp[8]     = "--";
  char code[4]     = "0";
  char updated[6]  = "--:--";
  bool valid       = false;
};

Weather wx;
uint32_t lastFetchMs = 0;
uint8_t  failCount   = 0;
```

## 7.3 主流程

```cpp
void setup() {
  Serial.begin(115200);
  u8g2.begin();
  u8g2.enableUTF8Print();
  u8g2.setFont(u8g2_font_wqy12_t_gb2312);
  u8g2.clearBuffer();
  u8g2.drawUTF8(0, 14, "天气助手启动中");
  u8g2.sendBuffer();

  connectWiFi();
  fetchWeather();
  drawScreen();
  lastFetchMs = millis();
}

void loop() {
  if (WiFi.status() != WL_CONNECTED) connectWiFi();

  if (millis() - lastFetchMs >= FETCH_INTERVAL_MS) {
    lastFetchMs = millis();
    if (fetchWeather()) failCount = 0;
    else if (failCount < 255) failCount++;
  }

  drawScreen();
  delay(1000);
}
```

> ⚠️ **别用 `delay(600000)`** —— 那样屏幕不能刷新、串口不能响应。用 `millis()` 做非阻塞调度。

## 7.4 带超时的联网

```cpp
bool connectWiFi() {
  WiFi.mode(WIFI_STA);
  WiFi.setAutoReconnect(true);
  WiFi.begin(WIFI_SSID, WIFI_PASS);

  uint32_t t0 = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - t0 < 20000) {
    delay(300);
    Serial.print('.');
  }
  Serial.println();
  return WiFi.status() == WL_CONNECTED;
}
```

> ⚠️ **20 秒超时必须有**：死等循环会让设备在 WiFi 故障时永远卡住，连 OTA 都救不回来。

## 7.5 HTTP 取数 + JSON 解析

`api.seniverse.com` 支持普通 HTTP（80 端口），不必开 TLS：

```cpp
void copyField(char* dst, size_t cap, JsonVariant v) {
  if (v.is<const char*>())      strlcpy(dst, v.as<const char*>(), cap);
  else if (v.is<int>() || v.is<float>())
    snprintf(dst, cap, "%s", v.as<String>().c_str());
}

bool fetchWeather() {
  if (WiFi.status() != WL_CONNECTED) return false;

  WiFiClient client;
  client.setTimeout(HTTP_TIMEOUT_MS / 1000);
  if (!client.connect(HOST, 80)) return false;

  String path = "/v3/weather/now.json?key=";
  path += API_KEY;
  path += "&location=" + String(CITY);
  path += "&language=zh-Hans&unit=c";

  client.print(String("GET ") + path + " HTTP/1.1\r\n" +
               "Host: " + HOST + "\r\n" +
               "Connection: close\r\n\r\n");

  uint32_t t0 = millis();
  while (!client.available()) {
    if (millis() - t0 > HTTP_TIMEOUT_MS) { client.stop(); return false; }
    delay(10);
  }

  client.find("\r\n\r\n");              // 流式跳过响应头

  DynamicJsonDocument doc(2048);       // 放堆上，别放栈
  DeserializationError err = deserializeJson(doc, client);
  client.stop();
  if (err) return false;

  JsonObject r0 = doc["results"][0];
  if (r0.isNull()) return false;

  copyField(wx.city, sizeof(wx.city), r0["location"]["name"]);
  copyField(wx.text, sizeof(wx.text), r0["now"]["text"]);
  copyField(wx.temp, sizeof(wx.temp), r0["now"]["temperature"]);
  copyField(wx.code, sizeof(wx.code), r0["now"]["code"]);

  const char* lu = r0["last_update"] | "";
  if (strlen(lu) >= 16) {
    snprintf(wx.updated, sizeof(wx.updated), "%c%c:%c%c",
             lu[11], lu[12], lu[14], lu[15]);
  }

  wx.valid = true;
  return true;
}
```

易错点：

1. `DynamicJsonDocument` 放堆上 —— ESP8266 栈只有 4KB，大 JSON 放栈必崩
2. `temperature` 是**字符串** `"26"`，不是数字
3. `client.find("\r\n\r\n")` 比 `readStringUntil` 省内存
4. `Connection: close` 必须有

## 7.6 渲染

```cpp
void drawScreen() {
  u8g2.clearBuffer();

  u8g2.setFont(u8g2_font_wqy12_t_gb2312);
  u8g2.drawUTF8(0, 12, wx.city);
  u8g2.drawStr(96, 12, wx.updated);
  u8g2.drawHLine(0, 15, 128);

  drawWeatherIcon(0, 26, wx.code);
  u8g2.setFont(u8g2_font_wqy16_t_gb2312);
  u8g2.drawUTF8(12, 40, wx.text);

  u8g2.setFont(u8g2_font_logisoso24_tn);
  String t = String(wx.temp) + "C";
  u8g2.drawStr(126 - u8g2.getStrWidth(t.c_str()), 42, t.c_str());

  u8g2.setFont(u8g2_font_6x10_tf);
  if (!wx.valid)                            u8g2.drawStr(0, 60, "NO DATA");
  else if (WiFi.status() != WL_CONNECTED)   u8g2.drawStr(0, 60, "OFFLINE (cached)");
  else if (failCount > 2)                   u8g2.drawStr(0, 60, "UPDATE FAILED");
  else                                      u8g2.drawStr(0, 60, "OK");

  u8g2.sendBuffer();
}
```

## 7.7 稳定性加固

「能用」和「敢长期插着」的分界线就三件事：

| 措施 | 原因 |
|:--|:--|
| WiFi 断连自动重连 | 路由器重启/信号抖动是常态 |
| HTTP 全程超时 | 没超时会永久卡死 |
| 缓存降级显示 | 断网时屏幕不该变白 |

**LittleFS 缓存**：成功一次就存下来，开机先读缓存再联网。

```cpp
#include <LittleFS.h>

void saveCache() {
  File f = LittleFS.open("/wx.json", "w");
  if (!f) return;
  StaticJsonDocument<256> d;
  d["city"] = wx.city; d["text"] = wx.text; d["temp"] = wx.temp;
  serializeJson(d, f);
  f.close();
}

bool loadCache() {
  File f = LittleFS.open("/wx.json", "r");
  if (!f) return false;
  StaticJsonDocument<256> d;
  if (deserializeJson(d, f)) { f.close(); return false; }
  copyField(wx.city, sizeof(wx.city), d["city"]);
  copyField(wx.text, sizeof(wx.text), d["text"]);
  copyField(wx.temp, sizeof(wx.temp), d["temp"]);
  wx.valid = true;
  f.close();
  return true;
}
```

`setup()` 里把 `LittleFS.begin(); loadCache();` 放在 `fetchWeather()` 之前。

**退避重试**：连续失败时拉长间隔，别把免费额度打爆。

```cpp
uint32_t interval() {
  if (failCount == 0) return FETCH_INTERVAL_MS;
  uint32_t backoff = FETCH_INTERVAL_MS * (1 << min<uint8_t>(failCount, 3));
  return min<uint32_t>(backoff, 30UL * 60UL * 1000UL);
}
```

**看门狗**：串口出现 `rst cause:4` 就是硬件看门狗复位，说明某处阻塞太久；`rst cause:2` 是正常上电。

## 7.8 NTP 对时

```cpp
#include <time.h>

void syncTime() {
  configTime(8 * 3600, 0, "ntp.aliyun.com", "ntp.tencent.com");
  struct tm t;
  if (getLocalTime(&t, 5000)) {
    Serial.printf("time %02d:%02d\n", t.tm_hour, t.tm_min);
  }
}
```

把 `drawScreen()` 里的 `wx.updated` 换成系统时间，就是真正的时钟了。

---

# 八、脱开 Arduino：ESP8266_RTOS_SDK

Arduino core 本身就是在 NONOS SDK 上包的一层。想真正控制时序、内存、任务调度，就得直接写 SDK。

ESP8266 有三条脱离 Arduino 的路：

| 路线 | 底层 | 框架 | 适合 |
|:--|:--|:--|:--|
| **ESP8266_RTOS_SDK** | FreeRTOS + lwIP | esp-idf 风格（`idf.py` / CMake） | **推荐**，v3.4 仍在维护 |
| ESP8266_NONOS_SDK | 无 OS，单事件循环 | `user_init()` + 定时器 | 最贴硬件，官方已停更 |
| AT 固件 | 无 | 主机 MCU 发 AT 指令 | 只当 WiFi 模块用 |

> ⚠️ **天花板提醒**：ESP8266 的 RF / PHY / WiFi 协议栈是闭源二进制（`libpp.a`、`librf.a`、`libnet80211.a`），无法像 STM32 那样从零写启动代码做到「真裸机」。不接 SDK 就没有 WiFi。

---

## 8.1 环境搭建

### 工具链

只支持 **GCC 8.4.0（esp-2020r3）**，旧版 SDK（< v3.0）才是 4.8.5。

| 平台 | 包 |
|:--|:--|
| Linux x64 | `xtensa-lx106-elf-gcc8_4_0-esp-2020r3-linux-amd64.tar.gz`（≈89MB）|
| Linux x86 | `...-linux-i686.tar.gz` |
| macOS | `...-macos.tar.gz`（≈95MB）|
| Windows | `...-win32.zip`（≈109MB）|

地址都在 `dl.espressif.com/dl/` 下。

### 装 SDK

```bash
mkdir -p ~/esp && cd ~/esp
git clone -b release/v3.4 --recursive https://github.com/espressif/ESP8266_RTOS_SDK.git
cd ESP8266_RTOS_SDK
./install.sh          # 下载工具链到 ~/.espressif + 建 python env
```

`install.sh` 干两件事：`idf_tools.py install`（拉工具链）和 `install-python-env`（建独立 venv）。国内慢的话可以先手动下工具链解压到 `~/.espressif/dist/`，脚本会认。

### 每次开终端

```bash
. ~/esp/ESP8266_RTOS_SDK/export.sh
```

必须 **source 不能执行**。它做的事：设 `IDF_PATH`、把 `xtensa-lx106-elf/bin` 加进 PATH、校验 python 依赖。

建议写进 `~/.bashrc`：

```bash
alias get_esp8266='. ~/esp/ESP8266_RTOS_SDK/export.sh'
```

### Python 依赖的坑

`requirements.txt` 里钉的是 **`cryptography>=2.1.4,<35`** 和 **`pyparsing<2.4.0`**。这两个版本上限都停在 2021 年前后，在 Python 3.11+ 上经常装不上（老版本没有对应 Python 版本的预编译 wheel，就得现场编译，而 `cryptography` 编译要 Rust + openssl-dev）。

解法：单独建一个 **Python 3.10** 的 venv 给 SDK 用，或者手动放宽版本约束：

```bash
python3.10 -m venv ~/.esp-venv && source ~/.esp-venv/bin/activate
pip install -r ~/esp/ESP8266_RTOS_SDK/requirements.txt
```

`export.sh` 会跑 `check_python_dependencies.py`，依赖不全直接 `return 1`，表现为「source 完什么都没变」。

### 验证

```bash
xtensa-lx106-elf-gcc --version     # 应显示 8.4.0
echo $IDF_PATH                     # 非空
```

---

## 8.2 工程结构

esp-idf 风格：**顶层 CMakeLists + main 组件**。

```
my_project/
├── CMakeLists.txt          # 顶层：include project.cmake + project(name)
├── Makefile                # 可选，走老 make 构建时用
├── sdkconfig               # menuconfig 产物，自动生成
├── sdkconfig.defaults      # 可选：默认配置，不进 git 的是 sdkconfig
└── main/
    ├── CMakeLists.txt      # idf_component_register(SRCS ...)
    ├── component.mk        # 老 make 构建用
    └── my_main.c
```

顶层 `CMakeLists.txt` 三行，顺序不能变：

```cmake
cmake_minimum_required(VERSION 3.5)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(my_project)
```

`main/CMakeLists.txt`：

```cmake
idf_component_register(SRCS "my_main.c"
                       INCLUDE_DIRS ".")
```

入口函数是 **`app_main()`**，不是 `main()` —— 它被 FreeRTOS 当做一个任务启动，返回后任务就结束了（所以要么死循环，要么自己建任务）。

### 两种构建系统

v3.4 同时保留老 make 和 CMake：

| | CMake（`idf.py`） | 老 Make |
|:--|:--|:--|
| 配置 | `idf.py menuconfig` | `make menuconfig` |
| 构建 | `idf.py build` | `make` |
| 烧录 | `idf.py -p /dev/ttyUSB0 flash` | `make flash` |
| 串口 | `idf.py -p /dev/ttyUSB0 monitor` | `make monitor` |
| 配置文件名 | `sdkconfig` | `sdkconfig` |

**建议用 `idf.py`**，命令和 ESP32 一致，以后换芯片不用重新学。老 make 在 v3.4 里已经属于兼容保留。

---

## 8.3 第一个工程：hello world

```bash
cp -r $IDF_PATH/examples/get-started/hello_world ~/esp/my_hello
cd ~/esp/my_hello
```

**例子不必待在 SDK 里**，拷到哪都行，靠 `IDF_PATH` 挂回 SDK。

### 必改的配置

`sdkconfig` 默认 **Flash 2MB**，NodeMCU 是 4MB，不改就是烧进去乱码 / 起不来。

```bash
idf.py menuconfig
```

| 菜单 | 项 | 改成 |
|:--|:--|:--|
| Serial flasher config | Flash size | **4 MB** |
| Serial flasher config | Default serial port | `/dev/ttyUSB0`（Windows 用 `COM3`）|
| Serial flasher config | Default baud rate | 921600（不稳降 115200）|
| Serial flasher config | Flash SPI mode | DIO（默认 QIO，多数板子也 OK）|
| Component config → FreeRTOS | Tick rate | 100（默认，够用）|

也可以直接写 `sdkconfig.defaults`，省得每次点：

```
CONFIG_ESPTOOLPY_FLASHSIZE_4MB=y
CONFIG_ESPTOOLPY_PORT="/dev/ttyUSB0"
CONFIG_ESPTOOLPY_BAUD_921600B=y
```

`menuconfig` 生成的是 `sdkconfig`，**`sdkconfig.defaults` 只在没有 `sdkconfig` 时生效**。改完 defaults 要 `rm sdkconfig` 再配一次。

### 构建与烧录

```bash
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

`Ctrl+]` 退出 monitor。

### main 代码

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_system.h"
#include "esp_spi_flash.h"

void app_main()
{
    printf("Hello world!\n");

    esp_chip_info_t chip_info;
    esp_chip_info(&chip_info);
    printf("This is ESP8266 chip with %d CPU cores, WiFi, ", chip_info.cores);
    printf("silicon revision %d, ", chip_info.revision);
    printf("%dMB %s flash\n", spi_flash_get_chip_size() / (1024 * 1024),
           (chip_info.features & CHIP_FEATURE_EMB_FLASH) ? "embedded" : "external");

    for (int i = 10; i >= 0; i--) {
        printf("Restarting in %d seconds...\n", i);
        vTaskDelay(1000 / portTICK_PERIOD_MS);
    }
    printf("Restarting now.\n");
    fflush(stdout);
    esp_restart();
}
```

### 分区表

默认 `PARTITION_TABLE_SINGLE_APP`，布局写死在 `components/partition_table/partitions_singleapp.csv`：

```
# Name,   Type, SubType, Offset,  Size
nvs,      data, nvs,     0x9000,  0x6000
phy_init, data, phy,     0xf000,  0x1000
factory,  app,  factory, 0x10000, 0xF0000   ← app 960KB
```

- **bootloader 固定烧在 0x0**（`CONFIG_BOOTLOADER_STORE_OFFSET`）
- app 从 `0x10000` 开始，上限 960KB —— ESP8266 固件很容易吃满
- 要 OTA 就切 `PARTITION_TABLE_TWO_OTA`，两个 960KB 槽 + otadata

想自定义就开 `PARTITION_TABLE_CUSTOM`，在工程根目录放 `partitions.csv`。

---

## 8.4 WiFi 连接

SDK 的 WiFi 是**事件驱动**的：`esp_wifi_start()` 之后立刻返回，连接过程通过事件回调推进。这和 Arduino 的 `WiFi.begin()` 阻塞式写法是完全不同的思路。

```c
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"
#include "esp_system.h"
#include "esp_log.h"
#include "esp_event.h"
#include "esp_wifi.h"
#include "nvs_flash.h"
#include "tcpip_adapter.h"
#include "lwip/err.h"
#include "lwip/sys.h"

#define WIFI_SSID  "myssid"
#define WIFI_PASS  "mypassword"
#define WIFI_RETRY 5

static const char *TAG = "wifi";
static EventGroupHandle_t s_wifi_events;
static int s_retry = 0;

#define WIFI_CONNECTED_BIT BIT0
#define WIFI_FAIL_BIT      BIT1

static void event_handler(void *arg, esp_event_base_t base,
                          int32_t id, void *data)
{
    if (base == WIFI_EVENT && id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();
    } else if (base == WIFI_EVENT && id == WIFI_EVENT_STA_DISCONNECTED) {
        if (s_retry < WIFI_RETRY) {
            esp_wifi_connect();
            s_retry++;
            ESP_LOGI(TAG, "retry %d", s_retry);
        } else {
            xEventGroupSetBits(s_wifi_events, WIFI_FAIL_BIT);
        }
    } else if (base == IP_EVENT && id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t *ev = (ip_event_got_ip_t *)data;
        ESP_LOGI(TAG, "got ip: %s", ip4addr_ntoa(&ev->ip_info.ip));
        s_retry = 0;
        xEventGroupSetBits(s_wifi_events, WIFI_CONNECTED_BIT);
    }
}

void wifi_init_sta(void)
{
    s_wifi_events = xEventGroupCreate();

    tcpip_adapter_init();                        // 注意：v3.4 还是这个，不是 esp_netif
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, event_handler, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, event_handler, NULL));

    wifi_config_t wc = { .sta = { .ssid = WIFI_SSID, .password = WIFI_PASS } };
    if (strlen(WIFI_PASS))
        wc.sta.threshold.authmode = WIFI_AUTH_WPA2_PSK;

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(ESP_IF_WIFI_STA, &wc));
    ESP_ERROR_CHECK(esp_wifi_start());

    EventBits_t bits = xEventGroupWaitBits(s_wifi_events,
                          WIFI_CONNECTED_BIT | WIFI_FAIL_BIT,
                          pdFALSE, pdFALSE, portMAX_DELAY);

    ESP_LOGI(TAG, bits & WIFI_CONNECTED_BIT ? "connected" : "failed");
}

void app_main()
{
    ESP_ERROR_CHECK(nvs_flash_init());   // WiFi 要读写 NVS，必须初始化
    wifi_init_sta();
}
```

### 几个必须知道的点

1. **`nvs_flash_init()` 必须先做**，否则 WiFi 起不来（要存校准参数）。NVS 满了会返回 `ESP_ERR_NVS_NO_FREE_PAGES`，得 erase 重来。
2. **`tcpip_adapter_init()`** —— v3.4 是这名字。官方 roadmap 说要迁到 `esp_netif`，但 v3.4 还没有。看到网上写 `esp_netif_init()` 的是 ESP32 的代码。
3. **`ESP_IF_WIFI_STA`** 是 v3.4 的枚举名；ESP-IDF v4 之后改成了 `WIFI_IF_STA`。照抄 ESP32 文档会编译不过。
4. **重连要自己写**。`WIFI_EVENT_STA_DISCONNECTED` 里不调 `esp_wifi_connect()` 就永久断开。生产代码还得加退避（别 5 次全在 100ms 内用完）。
5. **SSID / 密码写在 Kconfig 里**更规范 —— 建 `main/Kconfig.projbuild` 加 `config ESP_WIFI_SSID`，代码里用 `CONFIG_ESP_WIFI_SSID`，`menuconfig` 里改，不用动源码。

---

## 8.5 外设：GPIO / PWM / ADC

### GPIO（含中断）

```c
#include "driver/gpio.h"

void gpio_setup(void)
{
    gpio_config_t io = {
        .pin_bit_mask = (1ULL << 15) | (1ULL << 16),  // 输出
        .mode         = GPIO_MODE_OUTPUT,
        .pull_up_en   = 0,
        .pull_down_en = 0,
        .intr_type    = GPIO_INTR_DISABLE,
    };
    gpio_config(&io);

    io.pin_bit_mask = (1ULL << 4);                    // 输入 + 上升沿中断
    io.mode         = GPIO_MODE_INPUT;
    io.pull_up_en   = 1;
    io.intr_type    = GPIO_INTR_POSEDGE;
    gpio_config(&io);

    gpio_install_isr_service(0);
    gpio_isr_handler_add(4, my_isr, (void *)4);
}
```

`gpio_set_level(GPIO, 1)` / `gpio_get_level(GPIO)`。注意 **`pin_bit_mask` 是 `uint32_t`**（不是 ESP32 的 `uint64_t`），因为 ESP8266 只有 GPIO0~16。

中断里**不能**打日志，标准做法是往队列扔一个数，让任务去处理：

```c
static xQueueHandle q;
static void IRAM_ATTR my_isr(void *arg) {
    uint32_t pin = (uint32_t)arg;
    xQueueSendFromISR(q, &pin, NULL);
}
```

### PWM：LEDC vs 软件 PWM

ESP8266 有两套：

| | LEDC | `pwm.h`（软件） |
|:--|:--|:--|
| 实现 | 硬件 | 定时器 + GPIO 翻转 |
| 通道 | 8（高速 4 + 低速 4）| 8 |
| 精度 | 最高 13 bit | 依赖周期 |
| 占用 | 无 | 每个边沿一次中断 |
| 推荐 | ✅ | 需要相位控制时 |

LEDC 初始化分两步 —— **先配 timer，再配 channel**：

```c
#include "driver/ledc.h"

ledc_timer_config_t t = {
    .duty_resolution = LEDC_TIMER_13_BIT,
    .freq_hz         = 5000,
    .speed_mode      = LEDC_LOW_SPEED_MODE,
    .timer_num       = LEDC_TIMER_1,
};
ledc_timer_config(&t);

ledc_channel_config_t c = {
    .channel    = LEDC_CHANNEL_2,
    .duty       = 0,
    .gpio_num   = 4,
    .speed_mode = LEDC_LOW_SPEED_MODE,
    .hpoint     = 0,
    .timer_sel  = LEDC_TIMER_1,
};
ledc_channel_config(&c);
```

改占空比要**两步**，少一步就没反应（这是最常见的坑）：

```c
ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_2, 4096);
ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_2);   // 必须！
```

渐变要先装 fade 服务：

```c
ledc_fade_func_install(0);
ledc_set_fade_with_time(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_2, 8191, 3000);
ledc_fade_start(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_2, LEDC_FADE_NO_WAIT);
```

**高频通道（0/1/2/3 走 `LEDC_HIGH_SPEED_MODE`）在 WiFi 活跃时更稳**，低频通道会被 WiFi 抢占导致抖动。

### ADC

```c
#include "driver/adc.h"

adc_config_t cfg = { .mode = ADC_READ_TOUT_MODE, .clk_div = 8 };
adc_init(&cfg);

uint16_t v;
adc_read(&v);        // 0~1023，单位 1/1023 V
```

- **量程 0~1.0V**，和 Arduino 的 `analogRead` 一样，测 3.3V 必须分压
- `ADC_READ_TOUT_MODE` 读 A0 脚；`ADC_READ_VDD_MODE` 读供电电压（此时 A0 必须悬空）
- `clk_div` 范围 8~32，越小越快但噪声越大
- **WiFi 开着时 ADC 有噪声**，精度要求高就关 WiFi 或多次采样平均

---

## 8.6 日志与调试

`esp_log.h` 是 SDK 的核心工具，比 `printf` 强得多：

```c
#include "esp_log.h"
static const char *TAG = "app";

ESP_LOGE(TAG, "err %d", err);     // 红
ESP_LOGW(TAG, "warn");            // 黄
ESP_LOGI(TAG, "info %s", s);      // 绿
ESP_LOGD(TAG, "debug");           // 默认不编译
ESP_LOGV(TAG, "verbose");
```

- 输出带 **时间戳（毫秒）+ 级别 + TAG + 文件名:行号**
- `CONFIG_LOG_DEFAULT_LEVEL` 决定**编译进固件**的级别 —— Info 以下的不占 flash
- 运行时还能改：`esp_log_level_set("wifi", ESP_LOG_DEBUG)` 单独放开某个模块
- 看 WiFi 细节就 `esp_log_level_set("wifi", ESP_LOG_VERBOSE)`，比 Arduino 的 `WiFi.printDiag` 详细得多

### 崩溃回溯

`idf.py monitor` 会把崩溃地址自动翻译成函数名和行号：

```
Guru Meditation Error: Core  0 panic'ed (LoadProhibited)
PC      : 0x400d1234  in my_func at /home/u/proj/main/main.c:42
EXCVADDR: 0x00000000
Backtrace: 0x400d1234:0x3ffe8b60 0x400d5678:0x3ffe8b90
```

Arduino core 也有这个能力，但 SDK 的 `idf_monitor` 做得更完整（含 `make monitor` 的 `Ctrl+T Ctrl+H` 解码）。

### GDB

v3.4 的 `idf.py` **没有 `gdb` 子命令**（ESP-IDF 的 `idf.py gdb` 是后来才有的），只能直接调工具链：

```bash
xtensa-lx106-elf-gdb build/my_project.elf
(gdb) target remote :3333        # 需要先起 openocd / 或接 JTAG
```

`openocd` 对 ESP8266 支持一般，实际调试更常用「打日志 + 回溯」。真要断点得上 JTAG —— ESP8266 只复用 GPIO12~15 四根（MTDI/MTCK/MTMS/MTDO），接线很挤，NodeMCU 得飞线。日常不建议。

---

## 8.7 从 Arduino 迁移对照表

| Arduino | ESP8266_RTOS_SDK | 注意 |
|:--|:--|:--|
| `setup()` | `app_main()` 前段 | 没有「只跑一次」的语义，返回即退出 |
| `loop()` | `while (1)` 或 `xTaskCreate` | 建议拆成任务，别占满 |
| `delay(ms)` | `vTaskDelay(ms / portTICK_PERIOD_MS)` | **会调度其它任务**，不是忙等 |
| `millis()` | `xTaskGetTickCount() * portTICK_PERIOD_MS` | tick 默认 100Hz，精度 10ms |
| `pinMode` | `gpio_config()` | 位掩码一次配多个 |
| `digitalWrite` | `gpio_set_level()` | 返回 `esp_err_t` |
| `analogWrite` | LEDC 或 `pwm_*` | 精度好很多 |
| `analogRead` | `adc_read()` | 同样 0~1V |
| `Serial.print` | `ESP_LOGI` | 带级别和 TAG |
| `WiFi.begin` | 事件回调 + `esp_wifi_start` | **非阻塞**，要等事件 |
| `WiFi.localIP` | `IP_EVENT_STA_GOT_IP` 里拿 | 事件里才有 |
| `String` | 固定缓冲 + `snprintf` | SDK 没 `String`，是好事 |
| `ESP.getFreeHeap()` | `esp_get_free_heap_size()` | — |
| `ESP.restart()` | `esp_restart()` | — |
| `yield()` | `vTaskDelay(1)` 或 `taskYIELD()` | SDK 里不需要手动 yield |

**最大的思维转变**：Arduino 是「主循环 + 阻塞调用」，SDK 是「多任务 + 事件回调 + 非阻塞」。任何 `while` 死等都会让看门狗复位或 WiFi 断连。

---

## 8.8 踩坑清单

| 现象 | 原因 | 处理 |
|:--|:--|:--|
| `source export.sh` 后什么都没变 | python 依赖不全，脚本 `return 1` | 看它打印的报错，多半是 `cryptography` / `pyparsing` 版本 |
| 装依赖报找不到 wheel / 编译失败 | Python 版本太新，老包没有对应 wheel | 换 Python 3.10 venv |
| `idf.py` 命令找不到 | 没 source，或 source 失败 | 重开终端重新 `. export.sh` |
| 编译过但烧完乱码 / 起不来 | Flash size 还是 2MB | `menuconfig` → 4MB |
| 烧录报 `Failed to connect` | 波特率太高 / 没进下载模式 | 降 115200；手动按住 BOOT 再上电 |
| `undefined reference to esp_netif_init` | 抄了 ESP32 的代码 | v3.4 用 `tcpip_adapter_init()` |
| `WIFI_IF_STA` 未定义 | 同上 | 用 `ESP_IF_WIFI_STA` |
| WiFi 起来了但拿不到 IP | 只处理了 `WIFI_EVENT` 没处理 `IP_EVENT` | 两个都要注册 |
| 一启动就 `rst cause:4` | 看门狗，`app_main` 里死循环没让出 CPU | 循环里加 `vTaskDelay` |
| `nvs_flash_init` 返回错误 | NVS 分区满 / 分区表不匹配 | `idf.py erase_flash` 重烧 |
| `ledc_set_duty` 无效果 | 忘了 `ledc_update_duty` | 两句都要 |
| 堆不够 / malloc 失败 | ESP8266 只有 ~40KB 堆，WiFi 就吃掉一半 | 少用大缓冲，`esp_get_free_heap_size()` 盯着 |
| 频繁重启 + WiFi 断 | brownout，供电不足 | 换 USB 线 / 外接 3.3V |

---

## 8.9 学习路径

如果你已经在 A4 阶段（cpq 裸机）走过 STM32，这条路会顺很多 —— 中断、链接脚本、内存布局的概念是通的，新东西只有 FreeRTOS 的任务/队列/事件组和 lwIP。

建议顺序：

1. `hello_world` 跑通，会用 `idf.py` 全套命令
2. `peripherals/gpio` → 点灯 + 中断 + 队列
3. `peripherals/ledc` → PWM 呼吸灯
4. `wifi/getting_started/station` → 连上路由器
5. 自己写：**WiFi 连上后起一个 HTTP server 显示板子状态**（对应第五章 5.3 节的 Arduino 版）
6. `protocols/esp_http_client` → 抓一次心知天气，和 Arduino 版对比代码量
7. `protocols/sntp` → NTP 对时

做完这 7 步，再看 Arduino core 的源码（`cores/esp8266/`），会发现它就是对这些 SDK 调用的薄封装 —— 那时候你已经在封装之下了。

---

# 九、排障手册

| 症状 | 原因 | 处理 |
|:--|:--|:--|
| `Failed to connect to ESP8266` | 线无数据 / 驱动缺 / 波特率太高 | 换线 → 查驱动 → 降到 9600 |
| `Permission denied: /dev/ttyUSB0` | Linux 权限 | `usermod -aG dialout $USER` 后重登 |
| 烧录成功但串口无输出 | 波特率不对 / 没复位 | 改 115200，按 RST |
| WiFi 连不上 | **连了 5G** / 密码错 | 换 2.4G；`WiFi.printDiag(Serial)` |
| 一直重启 | 外设从板载 3.3V 取电过多 | 外接电源，共地 |
| 一直重启 + `rst cause:4` | 看门狗（阻塞）| 检查长 `delay` / 死等 |
| 屏幕全黑 | 地址错 / 接线错 | 跑 I2C 扫描；试 `0x3D` |
| 屏幕花屏雪花 | I2C 速率不稳 / 线太长 | 缩短线，`Wire.setClock(100000)` |
| 中文乱码/方块 | 用了 `drawStr` / 字体不含该字 | 改 `drawUTF8`；换 GB2312 字体；`.ino` 存 **UTF-8 无 BOM** |
| JSON 解析错 `InvalidInput` | 拿到的是 HTML（被劫持）| 打印原始 body 确认 |
| 温度显示 `--` | 字段是字符串不是数字 | 用本文 `copyField` |
| 每天下午数据停更 | 免费额度耗尽 | 拉长间隔；换 Open-Meteo |
| 长时间运行乱码/卡死 | `String` 导致堆碎片 | 改 `char[]` + `snprintf` |

**调试技巧**：解析前先把原始报文打出来

```cpp
String body = client.readString();
Serial.println(body.substring(0, 300));   // 只用调试，吃内存
```

---

# 十、坑位汇总

**硬件**
1. USB 线有「只能充电」的，烧录必须数据线
2. 板载 3.3V 只有 ~200mA，外设一律独立供电
3. A0 只能输入 1.0V，测高压必须分压
4. D3(GPIO0) 拉低进烧录模式、D8(GPIO15) 启动必须为低，接外设要避让
5. OLED VCC 别接 5V 且同时把 I2C 上拉到 5V —— IO 不耐 5V

**软件**
6. 代码用 GPIO 编号，不用 D 编号
7. `analogWrite` 是软件 PWM，占 CPU
8. 堆只有 ~40KB，少用 `String`，多用 `snprintf` + 固定缓冲
9. 循环里避免长 `delay`，用 `millis()` 调度
10. U8g2 要全缓冲 `_F_` 才能正常 `drawUTF8`
11. 中文源文件必须 UTF-8 无 BOM

**网络**
12. 全系**只支持 2.4GHz**
13. 公共 MQTT 服务（如 `test.mosquitto.org`）**不安全**，正式用自建

---

## 📎 一页速查

```
接线：OLED VCC→3V3  GND→GND  SCL→D1(GPIO5)  SDA→D2(GPIO4)
库：U8g2 + ArduinoJson v6
流程：WiFi → GET /v3/weather/now.json → JSON → U8g2
中文：drawUTF8 + u8g2_font_wqy16_t_gb2312 + enableUTF8Print()
稳定：超时 + 重连 + 缓存降级 + 退避重试
额度：免费 400 次/天，10 分钟一次刚好

脱离 Arduino（第八章）：
工具链 xtensa-lx106-elf-gcc 8.4.0 + SDK release/v3.4
. export.sh → idf.py build → -p /dev/ttyUSB0 flash monitor
必改：Flash size 4MB（默认 2MB）
入口 app_main()；WiFi 事件驱动 + tcpip_adapter_init()；改占空比 duty+update 两句
```

## 参考资料

- 太极创客：`taichi-maker.com`（ESP8266 物联网教程、心知天气、U8g2）
- ESP8266 Arduino Core 文档：`arduino-esp8266.readthedocs.io`
- 心知天气 API：`seniverse.com/docs`
- Open-Meteo：`open-meteo.com`
- U8g2 字体列表：`github.com/olikraus/u8g2/wiki/fntlistall`
- ESP8266_RTOS_SDK：`github.com/espressif/ESP8266_RTOS_SDK`（v3.4，本文第八章）
- ESP8266_RTOS_SDK 文档：`docs.espressif.com/projects/esp8266-rtos-sdk/en/latest/`
- ESP8266_NONOS_SDK：`github.com/espressif/ESP8266_NONOS_SDK`（v3.0.6）

---

## 相关文档

- `04-ESP32-CAM入门与图传实战` — 进阶到带摄像头的双核板
