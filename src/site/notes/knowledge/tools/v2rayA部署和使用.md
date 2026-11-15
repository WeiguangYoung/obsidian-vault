---
{"dg-publish":true,"permalink":"/knowledge/tools/v2rayA部署和使用/","title":"v2rayA 安装部署与使用","tags":["v2rayA","代理","网络","部署","排障","订阅","工具"],"dg-note-properties":{"title":"v2rayA 安装部署与使用","date":"2026-11-15","tags":["v2rayA","代理","网络","部署","排障","订阅","工具"]}}
---


# 🌐 v2rayA 安装部署与使用

> 本文记录 v2rayA 在 Linux 主机上的完整部署与运维要点：
> **安装升级 → 端口与目录 → 账号管理 → 订阅更新 → 数据层脏数据修复 → 排障速查**。
> 所有命令均来自实测，坑点都标注了症状与解决方式。

返回 [[首页\|🏠 首页]]

---

## 📑 目录

| 章 | 主题 |
|:--|:--|
| 一 | 它是什么、部署在哪 |
| 二 | 安装与升级（deb 包） |
| 三 | 目录与文件布局 |
| 四 | 端口一览 |
| 五 | 账号与登录 |
| 六 | 订阅管理 |
| 七 | 数据层：脏数据修复 |
| 八 | 服务管理 |
| 九 | 排障速查表 |
| 十 | 验证方法 |

---

## 一、它是什么、部署在哪

**v2rayA** 是 v2ray/xray 内核的 Web 管理面板，用 Go 写的单二进制。它做三件事：

1. **拉订阅**：把机场的订阅链接解析成节点列表
2. **跑内核**：按选择策略启动 v2ray-core，对外提供 socks5/http 代理
3. **管规则**：路由规则、PAC、DNS 劫持、TUN 透明代理

本机部署形态：

```
二进制品   /usr/bin/v2raya          （deb 包安装，systemd 管理）
服务       v2raya.service            （root 运行）
数据目录   /etc/v2raya/              （config.json + v2raya.db）
日志       /var/log/v2raya/v2raya.log
管理界面   http://<局域网IP>:2017
```

---

## 二、安装与升级（deb 包）

### 2.1 从 GitHub Releases 下载

v2rayA 的包名格式：`installer_debian_x64_<version>.deb`

```bash
VERSION=2.5.15
cd ~/downloads
curl -sSL -C - -o installer_debian_x64_${VERSION}.deb \
  "https://github.com/v2rayA/v2rayA/releases/download/v${VERSION}/installer_debian_x64_${VERSION}.deb"
```

### 2.2 ⚠️ 下载必须校验完整性（实测踩坑）

国内网络下 GitHub 下载**经常截断**。表现是文件大小对不上，`dpkg -i` 报：

```
dpkg-deb (subprocess): cannot copy archive member ... unexpected end of file or stream
dpkg: error processing archive ... cannot copy extracted data for './usr/share/v2raya/geoip.dat'
```

**安装前必须验证能完整解压**，否则会装坏包：

```bash
# 校验：能完整解包才算下载完成
rm -rf /tmp/debtest && mkdir -p /tmp/debtest
dpkg-deb -x installer_debian_x64_${VERSION}.deb /tmp/debtest && echo "OK 完整"
```

如果不 OK，用循环续传直到通过：

```bash
for i in $(seq 1 12); do
  timeout 100 curl -sSL --retry 2 -C - -o pkg.deb "<url>"
  rm -rf /tmp/debtest && mkdir -p /tmp/debtest
  dpkg-deb -x pkg.deb /tmp/debtest 2>/dev/null && { echo "COMPLETE"; break; }
  echo "chunk $i incomplete: $(stat -c%s pkg.deb)"
done
```

> `curl -C -` 续传是有效的：截断的包继续拉就能补齐，不需要每次从头下。

### 2.3 安装 / 升级

```bash
# 备份旧二进制（可选，便于回滚）
sudo cp /usr/bin/v2raya /tmp/v2raya.$(v2raya --version | awk '{print $1}').bak

# 安装（会自动覆盖旧版本，保留 /etc/v2raya 下的配置）
sudo dpkg -i installer_debian_x64_${VERSION}.deb

# 验证
v2raya --version
sudo systemctl restart v2raya
```

**关键**：`dpkg -i` 覆盖安装**不会删除** `/etc/v2raya/`，订阅、节点、路由规则都保留。但**数据库 schema 会在首次启动时自动迁移**，迁移可能引入问题（见第七章）。

### 2.4 ⚠️ 升级后必须验证内核真的起来了

`systemctl is-active v2raya` 返回 `active` **不代表代理可用**。v2rayA 面板和 v2ray 内核是两个进程：面板起来了，内核可能启动即崩溃。

```bash
# 1. 服务层
systemctl is-active v2raya

# 2. 内核层（关键！）
curl -sS http://127.0.0.1:2017/api/version | grep -oE '"lastKernelExit":"[^"]*"'
#   期望 "running"；若是 "stopped" 说明内核没起来

# 3. 端口层
ss -tlnp | grep -E ':2017[0-2]'
#   期望看到 20170 / 20171 / 20172 三个监听
```

三项全部通过才算升级成功。只满足第 1 项是最容易误判的状态 —— 界面正常、端口全无、所有走代理的程序超时。

崩溃原因可直接手动跑内核看：

```bash
# 内核二进制与版本
sudo /usr/bin/v2raya_core --version

# 手动跑一次，看真实报错
sudo /usr/bin/v2raya_core run -c /etc/v2raya/config.json 2>&1 | head -15
```

### 2.5 ⚠️ 升级后 geo 数据文件找不到（2.5.15 实测）

**症状**：面板 `active`，但 `lastKernelExit` 一直是 `stopped`，20170/20171 端口不存在。

手动跑内核报：

```
Failed to start: main: failed to load config files
> infra/conf: failed to build routing configuration
> common/geodata: illegal domain rule: geosite:geolocation-!cn
> common/geodata: failed to open geosite.dat
> stat /usr/bin/geosite.dat: no such file or directory
```

**根因**：内核默认在**自身二进制所在目录**找 `geosite.dat` / `geoip.dat`，而 deb 包把这两个文件装在 `/usr/share/v2raya/`。旧版本能自动定位，2.5.15 起需要显式指定。

**修复**：给服务加 systemd drop-in 注入资源目录环境变量（不改原 unit 文件）：

```bash
sudo mkdir -p /etc/systemd/system/v2raya.service.d
sudo tee /etc/systemd/system/v2raya.service.d/assetdir.conf >/dev/null <<'EOF'
[Service]
Environment=XRAY_LOCATION_ASSET=/usr/share/v2raya
EOF

sudo systemctl daemon-reload
sudo systemctl restart v2raya
```

验证注入生效：

```bash
systemctl show v2raya -p Environment --value
# 应包含 XRAY_LOCATION_ASSET=/usr/share/v2raya
```

修复后手动跑内核应能看到加载成功：

```
[dns module] geosite resolver set, rules reloaded with geosite expansion
```

> 用 drop-in 而不是直接改 `/lib/systemd/system/v2raya.service`：包升级会覆盖原 unit，drop-in 不会被冲掉。

---

## 三、目录与文件布局

| 路径 | 作用 | 权限 |
|:--|:--|:--|
| `/usr/bin/v2raya` | 主程序 | root:root 755 |
| `/etc/v2raya/config.json` | 运行时配置（inbounds/outbounds/DNS） | root:root 600 |
| `/etc/v2raya/v2raya.db` | **SQLite 数据库（核心）** | root:root 644 |
| `/var/log/v2raya/v2raya.log` | 运行日志 | — |
| `/etc/default/v2raya` | systemd 环境变量（可选） | — |

### 3.1 v2raya.db 的表结构

| 表/键 | 内容 |
|:--|:--|
| `servers` | 节点列表（address/port/protocol/**config_json**） |
| `subscriptions` | 订阅链接与最后更新时间 |
| `accounts` | 账号表（**新装时为空**） |
| `system_config` | 键值对，存所有运行时状态 |

`system_config` 里的关键键：

```
accounts:<username>          bcrypt 密码哈希（JSON 字符串）
system:jwtSecret             JWT 签名密钥
system:running               代理内核是否运行（"true"/"false"）
system:lastKernelExit        内核最后状态（"running"）
system:ports                 端口配置 {"socks5":20170,"http":20171,...}
system:setting               全局设置 JSON（自动更新、TUN、DNS 等）
outbound.proxy:setting       出站策略 {"type":"leastping","selected":"<节点链接>"}
outbound.proxy:connectedServers  已连接的节点列表
```

---

## 四、端口一览

| 端口 | 协议 | 用途 |
|:--|:--|:--|
| **2017** | HTTP | **管理界面 / API**（`0.0.0.0`，局域网可访问） |
| **20170** | SOCKS5 | 代理入口 |
| **20171** | HTTP | 代理入口（`curl -x` 用这个） |
| **20172** | HTTP | 带 PAC 的代理入口 |

```bash
# 查看监听
ss -tlnp | grep -E '2017|20170|20171|20172'
```

### 4.1 局域网访问

```bash
# 查本机局域网 IP
ip -4 addr show | grep -E 'inet .*(wlan|eth|enp)'

# 访问（把 <LAN_IP> 换成上一步查到的地址）
http://<LAN_IP>:2017
```

如果打不开，放行防火墙：

```bash
sudo ufw allow 2017/tcp
```

> 注意：`docker0`（形如 `172.17.x.1`）和 `cni0`（形如 `10.42.x.1`）是容器网卡，**不是**局域网地址。选带 `wlan`/`eth`/`enp` 的那个。

---

## 五、账号与登录

### 5.1 用户名从哪来

新装时 `accounts` 表是空的。首次设置密码会走「首次初始化」流程，把账号写进 `system_config` 的 `accounts:<username>` 键。

```bash
# 查看已有账号（需要 root）
sudo python3 -c "
import sqlite3
c=sqlite3.connect('/etc/v2raya/v2raya.db')
for r in c.execute(\"select count(*) from accounts\"): print('accounts表行数:', r[0])
for r in c.execute(\"select key,substr(value,1,30) from system_config where key like 'accounts:%'\"):
    print(r)
"
```

### 5.2 首次设置密码（API 方式，推荐）

`/api/account` 在「无账号」状态下不需要鉴权，可直接创建：

```bash
curl -sS -X POST http://127.0.0.1:2017/api/account \
  -H 'Content-Type: application/json' \
  -d "{\"username\":\"<用户名>\",\"password\":\"<密码>\"} "
```

返回 `{"code":"SUCCESS","data":{"token":"..."}}` 即成功。

### 5.3 登录

```bash
curl -sS -X POST http://127.0.0.1:2017/api/login \
  -H 'Content-Type: application/json' \
  -d "{\"username\":\"<用户名>\",\"password\":\"<密码>\"} "
```

返回的 `data.token` 就是 JWT，后续 API 调用带 `Authorization: Bearer <token>`。

### 5.4 忘记密码怎么办

**猜不出来**（存的是 bcrypt 哈希，不可逆）。两条路：

**A. 重置为已知密码（保留所有配置）**

```bash
# 生成 bcrypt 哈希（需要 pip install bcrypt）
python3 -c "import bcrypt; print(bcrypt.hashpw(b'新密码', bcrypt.gensalt(rounds=10, prefix=b'2a')).decode())"

# 写回数据库
sudo python3 -c "
import sqlite3, json
c=sqlite3.connect('/etc/v2raya/v2raya.db')
c.execute('update system_config set value=? where key=?', (json.dumps('<bcrypt哈希>'), 'accounts:<用户名>'))
c.commit(); print('updated', c.total_changes)
"
sudo systemctl restart v2raya
```

**B. 清空账号，回到首次初始化**

```bash
sudo python3 -c "
import sqlite3
c=sqlite3.connect('/etc/v2raya/v2raya.db')
c.execute(\"delete from system_config where key like 'accounts:%'\")
c.commit()
"
sudo systemctl restart v2raya
# 之后用 5.2 的 /api/account 重新设置
```

---

## 六、订阅管理

### 6.1 订阅存在哪

```bash
sudo python3 -c "
import sqlite3
c=sqlite3.connect('/etc/v2raya/v2raya.db')
for r in c.execute('select id, address, status from subscriptions'): print(r)
"
```

`status` 字段是**最后更新时间**（UTC）。如果它停在几个月前，说明订阅没更新过。

### 6.2 自动更新是关的（默认）

```json
{"subscriptionAutoUpdateMode": "none", "subscriptionAutoUpdateIntervalHour": 0}
```

建议在网页「设置」里开启自动更新（例如每 24 小时）。否则节点机场换了地址，你会一直用一个不存在的节点。

### 6.3 ⚠️ 订阅更新只能在网页上点

实测 v2rayA 2.5.15 的**公开 API 没有订阅更新端点**。二进制里能提取到的路由只有：

```
/api/account    /api/login    /api/message    /api/version
```

`/api/subscription*`、`/api/servers`、`/api/connections` 全部返回 **404**。订阅更新走的是前端内部方法，没有对应 REST 接口。

**所以更新订阅必须打开网页操作**：

```
http://<局域网IP>:2017  →  登录  →  「订阅」页  →  点更新
```

> 如果你在找自动化方案：直接用 curl 拉订阅链接解析出节点，然后走 API 导入节点（如果该版本有导入接口），比想办法调「更新订阅」更实际。

### 6.4 订阅内容手工验证

订阅拉不拉得到、内容对不对，可以脱离 v2rayA 单独验证：

```bash
SUB='https://xxx/members/getsub.php?service=xxx&id=xxx'
curl -sS --max-time 35 -o /tmp/sub.txt -w "code:%{http_code} size:%{size_download}\n" "$SUB"

# 多数机场返回 base64 编码的链接列表
base64 -d /tmp/sub.txt | head -c 300
base64 -d /tmp/sub.txt | grep -oE '^[a-z0-9]+://' | sort | uniq -c
```

正常输出类似：

```
      1 ss://
      5 vless://
```

如果一个都解析不出来，说明订阅过期、被限流（429）或格式变了，**不是 v2rayA 的问题**。

---

## 七、数据层：脏数据修复

### 7.1 症状：升级后节点全部空白

升级到 2.5.15 后，`servers` 表里节点数量正确，但关键字段全空：

```
id  sub_id  address  port  protocol
13    1       ''      0     ''
14    1       ''      0     ''
...
```

网页上节点列表显示空白，无法手动选点。**但代理仍然能通** —— 因为内核读的是 `config_json`，不读 `address` 列。

### 7.2 根因

**2.5.15 的写入 bug**：解析结果完整写进了 `config_json`，但顶层的 `address`/`port`/`protocol` 三列没同步写入。

```sql
-- config_json 里有完整信息
{"serverObj":{"ps":"<备注名>","add":"<节点IP>","port":"443","id":"<UUID>"}}
-- 但 address 列是空的
```

### 7.3 修复：从 config_json 回填

**不删数据重来**，直接把 json 里的值写回列。脚本：

```python
import sqlite3, json

DB = "/etc/v2raya/v2raya.db"
c = sqlite3.connect(DB)

for sid, cfg in c.execute("select id, config_json from servers").fetchall():
    if not cfg:
        continue
    d = json.loads(cfg)
    addr = port = proto = None
    so = d.get("serverObj")
    if isinstance(so, dict):
        addr = so.get("add") or so.get("server") or so.get("address")
        port = so.get("port")
        proto = so.get("protocol") or so.get("proto")
    try:
        port = int(port) if port not in (None, "") else 0
    except (TypeError, ValueError):
        port = 0
    proto = proto.strip().lower() if isinstance(proto, str) else ""
    if addr or port:
        c.execute("update servers set address=?, port=?, protocol=? where id=?",
                  (addr or "", port, proto, sid))

# 顺带清掉指向已失效节点的 selected
row = c.execute("select value from system_config where key='outbound.proxy:setting'").fetchone()
if row:
    s = json.loads(row[0])
    s["selected"] = ""
    c.execute("update system_config set value=? where key='outbound.proxy:setting'",
              (json.dumps(s),))

c.commit()
```

执行后：

```bash
sudo systemctl restart v2raya
```

### 7.4 为什么要清 `selected`

`outbound.proxy:setting` 的 `selected` 存的是**选中的节点链接**。订阅更新后旧节点消失，这个字段就指向一个不存在的节点。

```
selected: vless://...@<已下线IP>:443   ← 该 IP 已不在当前订阅里
当前订阅: <当前节点IP列表>
```

清空后，`leastping` 模式会自动选一个可用的；也可以在网页上手动选。

> 实测：即使 `selected` 指向失效节点，代理也可能仍然可用（因为运行时实际用的是内存中的自动选择结果）。但重启后可能失效，属于隐患，建议清掉。

### 7.5 操作前备份（必须）

```bash
sudo cp /etc/v2raya/v2raya.db /tmp/v2raya.db.bak.$(date +%Y%m%d%H%M%S)
```

回滚：

```bash
sudo systemctl stop v2raya
sudo cp /tmp/v2raya.db.bak.<时间戳> /etc/v2raya/v2raya.db
sudo systemctl start v2raya
```

> 如果数据库和 WAL 文件（`v2raya.db-wal`）不一致，回滚前先删掉 `-wal` 和 `-shm`。

---

## 八、服务管理

```bash
# 状态
sudo systemctl status v2raya
systemctl is-active v2raya

# 重启（改动配置后）
sudo systemctl restart v2raya

# 版本
v2raya --version

# 启动时间 / 重启次数
systemctl show v2raya -p ActiveEnterTimestamp -p NRestarts --value

# 日志
sudo tail -50 /var/log/v2raya/v2raya.log
sudo journalctl -u v2raya -n 50 --no-pager
```

### 8.1 服务单元内容

```
ExecStart=/usr/bin/v2raya --log-disable-timestamp
Environment=V2RAYA_LOG_FILE=/var/log/v2raya/v2raya.log
User=root
```

---

## 九、排障速查表

| 症状 | 原因 | 解决 |
|:--|:--|:--|
| `dpkg -i` 报 `unexpected end of file` | deb 包下载被截断 | `curl -C -` 续传，`dpkg-deb -x` 验证通过再装 |
| 登录报 `WRONG_CREDENTIALS` | 用户名/密码不对 | 查 `system_config` 里的 `accounts:*` 键，或用 5.4 重置 |
| 登录报 `{"first":true}` | 从未设置过密码 | 用 `/api/account` 首次设置（5.2） |
| `/api/subscription` 返回 404 | 该端点不存在（2.5.15） | 订阅更新只能在网页操作（6.3） |
| 网页节点列表全空白 | `servers` 表 address 列为空 | 第七章回填修复 |
| 代理不通但界面显示 running | `selected` 指向订阅里已不存在的节点 | 清空 `selected` 或在网页重选节点 |
| 订阅更新后节点数为空 | 订阅拉取被限流/过期 | 用 6.4 手工验证订阅原文 |
| 局域网打不开 2017 | 防火墙 | `sudo ufw allow 2017/tcp` |
| 改了配置不生效 | 未重启 | `sudo systemctl restart v2raya` |
| 面板 active 但 20170/20171 端口不存在 | 内核启动即崩溃，面板只管自己 | 查 `lastKernelExit`，手动跑 `v2raya_core` 看报错 |
| 内核报 `failed to open geosite.dat` | 内核在 `/usr/bin` 找 geo 文件，实际在 `/usr/share/v2raya` | 加 drop-in 注入 `XRAY_LOCATION_ASSET`（见 2.5） |
| 开了代理后 LLM/API 请求超时 | 程序把流量发往不存在的代理端口 | 先修内核；再给程序加 `no_proxy` 白名单 |
| 日志里大量 `geosite: expanded` | 正常信息级日志 | 忽略 |
| `no token present in request` | API 未带 token | 先 `/api/login` 拿 token，加 `Authorization: Bearer` |

---

## 十、验证方法

### 10.1 服务层

```bash
# 服务活着 + 内核在跑
curl -sS http://127.0.0.1:2017/api/version | python3 -c "
import sys,json
d=json.load(sys.stdin)['data']
for k in ['version','coreVersion','lastKernelExit','serviceValid','hasAccounts']:
    print(f'  {k}: {d.get(k)}')
"
```

期望：

```
  version: 2.5.15
  coreVersion: 2.5.15
  lastKernelExit: running
  serviceValid: True
  hasAccounts: True
```

### 10.2 代理层（真正重要的验证）

```bash
# HTTP 代理
curl -sS -x http://127.0.0.1:20171 -o /dev/null \
  -w "youtube:%{http_code} %{time_total}s\n" --max-time 20 https://www.youtube.com

# SOCKS5 代理
curl -sS -x socks5h://127.0.0.1:20170 -o /dev/null \
  -w "google:%{http_code} %{time_total}s\n" --max-time 20 https://www.google.com

# 直连对照（确认问题不在本地网络）
curl -sS -o /dev/null -w "baidu:%{http_code}\n" --max-time 10 https://www.baidu.com
```

三个都 200 才算彻底正常。如果直连是 200 而代理超时，问题在节点侧，不在本机网络。

### 10.3 数据层

```bash
sudo python3 -c "
import sqlite3
c=sqlite3.connect('/etc/v2raya/v2raya.db')
print('节点总数:', c.execute('select count(*) from servers').fetchone()[0])
print('空地址:', c.execute(\"select count(*) from servers where address=''\").fetchone()[0])
for r in c.execute('select id,address,port,protocol from servers order by id'):
    print(' ', r)
"
```

「空地址」必须是 0，否则回到第七章修复。

---

## 附：本次升级实录

```
2026-11-15
2.5.7 → 2.5.15

遇到的问题：
1. GitHub 下载首次只有 26MB（实际 33MB），dpkg -i 解压失败 → curl -C - 续传解决
2. accounts 表为空，不知道用户名密码 → 用户名记录在 system_config 的 accounts:* 键
3. /api/account 首次初始化方式设密码（非 admin/admin）
4. 订阅更新后 servers.address 全空 → config_json 回填修复
5. selected 指向订阅里已下线的旧节点 → 清空后自动重选
6. 公开 API 无订阅更新端点 → 必须在网页点

最终状态：2.5.15 运行正常，代理 youtube/google/github 全部 200
```

---

> 🦐 虾管家 · 2026-11-15
