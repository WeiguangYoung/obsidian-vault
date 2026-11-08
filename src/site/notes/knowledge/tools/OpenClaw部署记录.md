---
{"dg-publish":true,"permalink":"/knowledge/tools/OpenClaw部署记录/","tags":["OpenClaw","部署记录","钉钉","systemd"],"dg-note-properties":{"date":"2026-11-07","tags":["OpenClaw","部署记录","钉钉","systemd"]}}
---


# OpenClaw 部署记录（独立实例）

> 主机：局域网内一台 Ubuntu 22.04 桌面机
> 日期：2026-11-07
> 目标：部署一个**完全独立**的 OpenClaw 实例，局域网可访问、systemd 托管、接入钉钉

---

## 一、主机初始状态

| 项 | 值 |
|---|---|
| 主机名 | `home-pc` |
| 系统 | Ubuntu 22.04.5 LTS |
| 内核 | 6.8.0-40-generic |
| 架构 | x86_64 |
| CPU | 12 核 |
| 内存 | 31 GB（可用 29 G） |
| 磁盘 | 916 GB NVMe，已用 8 G，剩 861 G |
| 局域网 IP | `<LAN_IP>` |
| 用户 | `<USER>` |

**初始缺失：**

- ❌ Node.js / npm
- ❌ git、curl、gcc、make、jq
- ❌ openssh-server（无法 SSH 登录）
- ❌ net-tools
- ✅ wget 1.21.2、python3 3.10.12、tar、unzip 已自带

---

## 二、网络与 SSH 打通

### 1. 连通性排查

```bash
# 从同局域网的另一台机器发起
ping -c 20 <LAN_IP>
# 结果：0% 丢包，平均 64.6 ms，最高 159.5 ms（WiFi 抖动偏大）
```

### 2. SSH 服务未启动

```bash
# 22 端口测试
timeout 8 bash -c 'cat < /dev/null > /dev/tcp/<LAN_IP>/22'
# Connection refused → 主机在线但 sshd 未运行
```

### 3. 安装并启动 openssh-server

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

**验证：**

| 检查 | 结果 |
|---|---|
| 22 端口 | ✅ 可连 |
| Banner | `SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.18` |
| 认证方式 | `publickey,password` |

### 4. 登录方式

```bash
# 本机无 SSH 密钥，暂用密码认证
sshpass -p '<密码>' ssh <USER>@<LAN_IP>
```

> 凭证见 `00-凭证管理.md` 中「局域网主机」一节。

### 5. net-tools 安装

```bash
sudo apt install -y net-tools
```

**用途**：`ifconfig` / `netstat` / `route` 等传统网络排查命令，部分脚本仍依赖。

**已安装版本**：`net-tools 1.60+git20181103.0eebece-1ubuntu5.4`

---

## 三、系统依赖安装

```bash
sudo apt update
sudo apt install -y git curl build-essential jq
```

| 工具 | 版本 | 用途 |
|---|---|---|
| git | 2.34.1 | 代码同步 |
| curl | 7.81.0 | 网络请求 |
| gcc | 11.4.0 | 原生模块编译 |
| make | 4.3 | 构建 |
| jq | 1.6 | JSON 处理 |

---

## 四、Node.js 安装（踩坑记录）

### 坑 1：curl | sudo tee 污染 apt 源

**错误写法：**

```bash
echo 'deb [...]' | echo "$PW" | sudo -S tee /etc/apt/sources.list.d/nodesource.list
```

**后果**：密码被写进了源文件，报错：

```
E: Malformed line 1 in source list /etc/apt/sources.list.d/nodesource.list (type)
```

**正确做法**：写成独立脚本文件再 sudo 执行

```bash
cat > /tmp/setup-ns.sh <<'EOS'
#!/bin/bash
set -e
rm -f /etc/apt/sources.list.d/nodesource.list
mkdir -p /etc/apt/keyrings
gpg --batch --yes --dearmor -o /etc/apt/keyrings/nodesource.gpg < /tmp/ns.key
chmod 644 /etc/apt/keyrings/nodesource.gpg
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_24.x nodistro main" \
  > /etc/apt/sources.list.d/nodesource.list
EOS
chmod +x /tmp/setup-ns.sh
echo "$PW" | sudo -S /tmp/setup-ns.sh
```

### 坑 2：Node 22 版本不够

OpenClaw 2026.11.1 要求 **Node ≥ 24.16.0 < 25**：

```
npm error [openclaw] error: this OpenClaw release requires Node >=24.16.0 <25 || >=26.1.0.
npm error [openclaw] detected Node 22.23.4
```

**解决**：切到 NodeSource 24.x 源

```bash
# 把 nodesource.list 里的 node_22.x 改成 node_24.x
sudo apt-get update -qq
sudo apt-get install -y nodejs
```

**最终版本：**

| 组件 | 版本 |
|---|---|
| Node.js | **v24.22.2** |
| npm | 11.19.1 |

---

## 五、npm 换源与全局路径

### 1. 权限问题

默认全局前缀是 `/usr`，非 root 安装会失败：

```
npm error path: '/usr/lib/node_modules/openclaw'
npm error The operation was rejected by your operating system.
```

### 2. 改成用户级前缀 + 淘宝镜像

```bash
mkdir -p /home/<USER>/.npm-global
npm config set prefix /home/<USER>/.npm-global --global
npm config set registry https://registry.npmmirror.com --global

# PATH 持久化
cat > /etc/profile.d/npm-global.sh <<'EOP'
export PATH=/home/<USER>/.npm-global/bin:$PATH
EOP
```

**效果对比：**

| 阶段 | 源 | 耗时 |
|---|---|---|
| 换源前 | registry.npmjs.org | 3 分钟未完成 |
| **换源后** | **registry.npmmirror.com** | **6-7 秒** |

### 3. 允许安装脚本

```bash
npm config set allow-scripts="@google/genai,esbuild,protobufjs,openclaw" --location=user
npm install -g openclaw@latest
```

**结果**：`added 360 packages`，`OpenClaw 2026.11.1 (4ffdd20)`

---

## 六、Gateway 配置（局域网访问）

### 1. 生成 Token

```bash
TOK=$(openssl rand -hex 24)
echo "$TOK" > ~/.openclaw-lan-token.txt
chmod 600 ~/.openclaw-lan-token.txt
```

### 2. 配置项

```bash
openclaw config set gateway.bind lan
openclaw config set gateway.auth.mode token
openclaw config set gateway.auth.token "$TOK"
```

| 配置 | 值 |
|---|---|
| `gateway.bind` | `lan`（监听 0.0.0.0） |
| `gateway.auth.mode` | `token` |
| `gateway.port` | `18789` |

### 3. 验证

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://<LAN_IP>:18789/
# HTTP 200

curl -s http://<LAN_IP>:18789/ | grep -o "<title>[^<]*</title>"
# <title>OpenClaw Control</title>
```

**访问地址：`http://<LAN_IP>:18789/`**

---

## 七、模型配置（与本机保持一致）

### Provider 配置

| Provider | Base URL | 模型 |
|---|---|---|
| **deepseek** | `https://api.deepseek.com/v1` | `deepseek-v4-flash` / `deepseek-v4-pro` / `deepseek-reasoner` |
| **siliconflow** | `https://api.siliconflow.cn/v1` | `deepseek-ai/DeepSeek-V4-Flash` |

### Agent 配置

```json5
agents: {
  defaults: {
    model: { primary: "deepseek/deepseek-v4-flash" },
    compaction: { mode: "safeguard" },
    timeoutSeconds: 1200,
    maxConcurrent: 4,
    subagents: { maxConcurrent: 8 },
    modelPolicy: {
      allow: [
        "siliconflow/deepseek-ai/DeepSeek-V4-Flash",
        "deepseek/deepseek-reasoner",
        "deepseek/deepseek-v4-flash"
      ]
    },
    thinkingDefault: "off",
  },
  entries: {
    main: {
      identity: { name: "虾助手", emoji: "🦐", theme: "实用、高效、友好" },
    },
  },
}
```

### 应用方式

```bash
openclaw config patch --file /tmp/openclaw-models.json5 --dry-run   # 先验证
openclaw config patch --file /tmp/openclaw-models.json5             # 实际应用
```

**结果**：`Applied 27 config update(s)`，无需重启

### 模型实测

```bash
openclaw agent -m "只回复这一句：OpenClaw 模型配置成功"
# 输出：OpenClaw 模型配置成功
```

---

## 八、systemd 托管与开机自启

### 1. 停止临时进程

```bash
# 之前是 nohup 启动
kill 7880
```

### 2. 安装服务

```bash
openclaw gateway install \
  --port 18789 \
  --token "$TOK" \
  --allow-unconfigured \
  --runtime-path /usr/bin/node \
  --force
```

**生成位置**：`~/.config/systemd/user/openclaw-gateway.service`

### 3. 服务单元关键配置

```ini
ExecStart=/usr/bin/node --max-old-space-size=8192 \
  /home/<USER>/.npm-global/lib/node_modules/openclaw/dist/index.js \
  gateway --port 18789 --allow-unconfigured
Restart=always
RestartSec=5
WantedBy=default.target
```

### 4. 启用持久化

```bash
systemctl --user daemon-reload
systemctl --user enable --now openclaw-gateway

# 关键：开启 linger，未登录也能运行
sudo loginctl enable-linger <USER>
```

### 5. 验证

| 检查 | 结果 |
|---|---|
| 服务状态 | ✅ `active (running)` |
| 开机自启 | ✅ `enabled` |
| linger | ✅ `Linger=yes` |
| 端口 | ✅ `0.0.0.0:18789` |
| 局域网访问 | ✅ HTTP 200（0.28s） |

**管理命令：**

```bash
systemctl --user status openclaw-gateway
systemctl --user restart openclaw-gateway
journalctl --user -u openclaw-gateway -f
```

---

## 九、钉钉插件（未完成）

### 1. 插件安装

```bash
openclaw plugins install @soimy/dingtalk --force
```

**为什么需要 `--force`**：该包来自 npm registry，未经 ClawHub 审核：

```
WARNING - Installing plugin from npm registry: @soimy/dingtalk
This source is outside ClawHub review and trust metadata.
```

### 2. 包信息核查

| 项 | 值 |
|---|---|
| 版本 | 3.9.1 |
| 作者 | YM Shen <soimy@163.com> |
| 仓库 | `github.com/soimy/openclaw-channel-dingtalk` |
| 许可证 | MIT |
| 月下载量 | 3,645 |
| 最近更新 | 2026-10-27 |

**结论**：插件已安装成功（`Applied in Gateway generation 3`）

### 3. 配置字段确认

| 字段 | 是否必需 | 说明 |
|---|---|---|
| `clientId` | ✅ 必需 | 钉钉 App Key |
| `clientSecret` | ✅ 必需 | 钉钉 App Secret |
| `robotCode` | ❌ 不需要 | schema 中不存在，运行时自动推导 |
| `corpId` | ❌ 已废弃 | Stream 模式不再读取 |
| `agentId` | ❌ 已废弃 | 同上 |

### 4. 配置尝试与失败

```bash
openclaw config set channels.dingtalk.enabled true
openclaw config set channels.dingtalk.clientId "***"
openclaw config set channels.dingtalk.clientSecret "***"
```

**失败原因**：写入的是字面量 `***`（3 个字符），不是真实凭证。

**日志证据：**

```
Config last-known-good promotion skipped: redacted secret placeholder at channels.dingtalk.clientId
[dingtalk] [default] Connection attempt 1 failed: Socket unavailable after connect
```

**验证存储值：**

```bash
python3 -c '
import json
d=json.load(open("/home/<USER>/.openclaw/openclaw.json"))
dt=d["channels"]["dingtalk"]
print(len(dt["clientId"]))'   # 输出 3
```

### 5. 待办

需要用真实凭证重新写入，三种方式：

| 方式 | 说明 |
|---|---|
| **A. secrets 工具** | 值进密钥库，不进聊天记录 |
| **B. 直接贴到聊天** | 明文会留在聊天记录 |
| **C. 手动在目标机执行** | `openclaw config set ...` |

### 6. 关于钉钉机器人类型的说明

**凭证文档第六章现有的是「自定义机器人（群机器人）」：**

```
ID:     <DINGTALK_APPKEY>
Secret: <DINGTALK_APPSECRET>
```

> 实际值见 `00-凭证管理.md` 第六章。

| 类型 | 能力 |
|---|---|
| **自定义机器人**（webhook） | 只能**单向推送**，不能接收消息 |
| **企业内部应用** | 支持 Stream 双向收发（聊天必需） |

如果要做对话，需要到钉钉开放平台建**企业内部应用**，取 AppKey / AppSecret。

---

## 十、最终状态

| 项 | 值 |
|---|---|
| **访问地址** | `http://<LAN_IP>:18789/` |
| **OpenClaw 版本** | 2026.11.1 (4ffdd20) |
| **Node** | v24.22.2 |
| **npm** | 11.19.1（淘宝源） |
| **默认模型** | `deepseek/deepseek-v4-flash` |
| **身份** | 虾助手 🦐 |
| **托管** | systemd 用户服务，开机自启 |
| **认证** | token（`~/.openclaw-lan-token.txt`） |
| **钉钉** | ⚠️ 插件已装，凭证未正确写入 |
| **飞书** | 未接入 |

---

## 十一、经验总结

### 1. sudo + 管道容易出事

`echo PW | sudo -S tee file` 会把密码写进文件。**改用独立脚本**：

```bash
cat > /tmp/x.sh <<'EOS'
#!/bin/bash
set -e
# 所有 sudo 操作
EOS
echo "$PW" | sudo -S /tmp/x.sh
```

### 2. OpenClaw 对 Node 版本要求严格

2026.11.1 要求 **Node ≥ 24.16**，Node 22 会被 preinstall 脚本直接拒绝。装之前先看 `package.json` 的 `engines`。

### 3. npm 全局安装必须改前缀

默认 `/usr` 需要 root。改成 `~/.npm-global` 即可普通用户安装。

### 4. 国内必须换 npm 源

`registry.npmmirror.com` 比官方源快 30 倍以上（3 分钟 vs 6 秒）。

### 5. systemd 用户服务必须开 linger

`loginctl enable-linger` 是关键一步——不开的话，用户不登录服务就不会启动。

### 6. rclone FUSE 挂载有缓存

**这个问题坑过一次**：COS 远端 14:02 更新了凭证文档，但通过 `/home/<USER>/cos` 挂载点读到的是 13:19 的缓存。

**正确做法**：直接读远端

```bash
rclone cat "cos:openclaw-1300723851/obsidian/notes/1-工作/00-凭证管理.md"
```

或用 `rclone lsjson` 看真实 `ModTime`。

### 7. 处理敏感值时别在命令里脱敏

在生成命令时把真实值替换成 `***`，会把占位符写进配置文件。**敏感值应走 secrets 工具**，或在最后一步才替换。
