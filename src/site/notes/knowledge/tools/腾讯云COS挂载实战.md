---
{"dg-publish":true,"permalink":"/knowledge/tools/腾讯云COS挂载实战/","tags":["DevOps","对象存储","腾讯云","COS","rclone","FUSE"],"dg-note-properties":{"date":"2026-09-27","updated":"2026-09-27","tags":["DevOps","对象存储","腾讯云","COS","rclone","FUSE"]}}
---


# 腾讯云 COS 挂载实战（rclone）

> 把腾讯云对象存储（COS）通过 **rclone + FUSE** 挂到本地目录，像本地磁盘一样读写；写入经本地 VFS 缓存后异步上传。
> 本文记录**生产在用的实际方案**：二进制安装 + systemd 常驻 + `--vfs-cache-mode full`。
> 适用场景：把对象存储当「云盘 / 同步盘」用（笔记库、备份、静态资源），机器无需额外块存储。

## 一、为什么要挂载

对象存储（S3 兼容）只有 HTTP API，不能像磁盘一样 `open/read/write`。挂载的价值：

- **对应用透明**：应用/编辑器把 COS 当普通目录读写，无需改代码或引 SDK
- **免运维**：不占机器块存储，容量按桶弹性
- **多端同步**：本地写的文件异步上传，换机器/换终端可见

代价（要清楚）：

- **强一致性与延迟**：随机写、大量小文件、边写边改的大文件，都比本地盘慢且易出错（见第六节）
- **缓存即真相**：写文件先进本地缓存，**上传完成前云端还没这份数据**

## 二、架构

```
本地机器                                 腾讯云
┌────────────────────────┐              ┌──────────────────┐
│  应用 / 编辑器          │              │   COS 桶          │
│      │ read/write       │              │  (S3 兼容 API)    │
│      ▼                  │   HTTPS      │                  │
│  /data1 (FUSE 挂载点)   │ ───────────▶ │  cos.<region>.   │
│      │                  │   对象读写    │  myqcloud.com    │
│  rclone mount 进程      │              │                  │
│      │ 本地 VFS 缓存     │              │                  │
│   /tmp/rclone/vfs/...   │              │                  │
└────────────────────────┘              └──────────────────┘
```

- **rclone**：把 S3 兼容 API 翻译成文件系统语义（POSIX 子集），通过 FUSE 暴露挂载点
- **VFS 缓存**：`--vfs-cache-mode full` 下，读/写都在本地缓存目录落地，读写完成后再与远端同步

## 三、安装 rclone（二进制）

生产推荐**官方二进制**（比发行版包新、可控）：

```bash
curl -O https://downloads.rclone.org/rclone-current-linux-amd64.zip
unzip rclone-current-linux-amd64.zip
cd rclone-*-linux-amd64
sudo cp rclone /usr/bin/
sudo chmod 755 /usr/bin/rclone
rclone version          # 验证
```

> 境内直连 `downloads.rclone.org` 不稳时，用 GitHub 代理（如 `ghfast.top`）或从能通网的机器中转。

> ⚠️ **FUSE 版本坑（Debian/Ubuntu 必踩，2026-09 实测）**：rclone ≥ 1.53 走 **FUSE 3**，需要 **`fusermount3`**。
> 老发行版（如 Ubuntu 20.04）默认只装 FUSE 2，挂载直接失败：
>
> ```
> Fatal error: failed to mount FUSE fs: fusermount: exec: "fusermount3": executable file not found in $PATH
> ```
>
> 解决：`sudo apt install -y fuse3`（RHEL 系 `dnf install fuse3`）。
> ⚠️ 别被 `fusermount -V` 误导 —— 它显示 `2.9.9`（FUSE 2）照样报上面这个错，必须确认 **`fusermount3`** 存在。

## 四、配置远程（rclone config）

### 4.1 交互式

```bash
rclone config
# n) New remote → name: cos → Storage: s3 → provider: TencentCOS
# env_auth: false → 填 access_key_id / secret_access_key
# endpoint: cos.ap-shanghai.myqcloud.com（按桶所在地域）
# acl: private
```

### 4.2 直接写配置文件（推荐，便于版本化/复用）

`~/.config/rclone/rclone.conf`：

```ini
[cos]
type = s3
provider = TencentCOS
env_auth = false
access_key_id = <AKIDxxxxxxxx>
secret_access_key = <xxxxxxxx>
endpoint = cos.ap-shanghai.myqcloud.com
acl = private
```

> ⚠️ **权限收紧**：该文件含 AK/SK 明文，务必 `chmod 600`。
> ⚠️ 推荐用**子账号 + 最小权限**（仅目标桶的读写），不要用主账号密钥。

### 4.3 验证连接

```bash
rclone lsd cos:                          # 列桶
rclone lsl cos:<bucket>/                 # 列对象
```

## 五、挂载

### 5.1 手动挂载 / 卸载

```bash
mkdir -p /data1
rclone mount cos:<bucket> /data1 \
  --vfs-cache-mode full \
  --allow-other

# 卸载
fusermount -u /data1
```

### 5.2 关键参数

| 参数 | 作用 |
|:---|:---|
| `--vfs-cache-mode full` | 读+写都走本地缓存，兼容性最好（支持 append、随机写、`O_TRUNC` 等）。**代价**：大文件双份占用本地磁盘 |
| `--allow-other` | 允许 root 以外的用户访问挂载点（默认仅挂载者可见）|
| `--vfs-cache-max-size` | 限制本地缓存上限（如 `10G`），防缓存撑爆磁盘 |
| `--vfs-cache-max-age` | 缓存最长保留时间（如 `24h`）|
| `--buffer-size` / `--dir-cache-time` | 调吞吐与目录缓存时长 |
| `--cache-dir` | 自定义缓存目录（默认在 `/tmp/rclone` 或 `~/.cache/rclone`）|

> 缓存模式四档：`off`（直读，不支持写入）→ `minimal` → `writes` → `full`（最兼容、最占空间）。需要**随机写/数据库/编辑器**，选 `full`。

### 5.3 systemd 常驻（开机自挂）

`/etc/systemd/system/rclone-cos.service`：

```ini
[Unit]
Description=Rclone mount COS to /data1
After=network.target

[Service]
Type=notify
ExecStart=/usr/bin/rclone mount cos:<bucket> /data1 --vfs-cache-mode full --allow-other
ExecStop=/bin/fusermount -u /data1
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now rclone-cos.service
systemctl status rclone-cos.service
```

> `Type=notify`：rclone 挂载就绪后会通知 systemd，避免「服务已起但挂载还没好」的竞态。
> `Restart=always` + `RestartSec=5`：网络抖动/进程崩溃自动重挂。

### 5.4 验证

```bash
mount | grep fuse.rclone          # 看到 cos:<bucket> on /data1 type fuse.rclone
findmnt /data1
```

### 5.5 无 root 场景：整套落到用户级（实测可用）

若机器上拿不到 root、或不想动系统目录，这套方案可以**完全不碰 `/etc` 和 `/usr`**：

| 系统级做法 | 用户级替代 |
|:---|:---|
| `sudo cp rclone /usr/bin/` | `install -m 755 rclone ~/.local/bin/rclone` |
| `/etc/systemd/system/rclone-cos.service` | `~/.config/systemd/user/rclone-cos.service` |
| `systemctl enable` | `systemctl --user enable --now` |
| 挂载点 `/data1` | `~/cos` |
| `--allow-other` | **去掉**（`/etc/fuse.conf` 的 `user_allow_other` 默认是注释态，改它要 root）|

前置条件：用户级服务能常驻 —— `loginctl show-user $USER | grep Linger` 应为 `Linger=yes`（否则需 `sudo loginctl enable-linger $USER`）。

`~/.config/systemd/user/rclone-cos.service`：

```ini
[Unit]
Description=Rclone mount COS
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=%h/.config/rclone/cos.env
ExecStart=%h/.local/bin/rclone mount cos:${COS_BUCKET} ${COS_MOUNT} \
  --vfs-cache-mode full \
  --vfs-cache-max-size 20G \
  --vfs-cache-max-age 24h \
  --cache-dir %h/.cache/rclone \
  --dir-cache-time 12h \
  --buffer-size 32M \
  --log-level INFO
ExecStop=/usr/bin/fusermount3 -u ${COS_MOUNT}
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

配套 `~/.config/rclone/cos.env`（`chmod 600`）把桶名/挂载点外置，以后改挂载点不用动 unit：

```ini
COS_BUCKET=<bucket>-<appid>
COS_MOUNT=/home/<user>/cos
```

```bash
chmod 600 ~/.config/rclone/rclone.conf ~/.config/rclone/cos.env
systemctl --user daemon-reload
systemctl --user enable --now rclone-cos.service
```

> 用户级 unit 里 `%h` = 家目录，`${VAR}` 由 `EnvironmentFile` 提供，都可直接使用。
> `ExecStop` 用 `fusermount3`（FUSE 3）而不是 `fusermount`。

## 六、坑与注意事项

### 6.1 边写边改的大文件会上传失败（重点）

`--vfs-cache-mode full` 下，文件在本地缓存写完后才上传。若**上传过程中源文件还在被改写**（如下载中的 APK、正在写的日志），rclone 会：

```
Failed to copy: can't copy - source file is being updated (size changed from ...)
corrupted on transfer: sizes differ src(...) 325058560 vs dst(...) 323485696
```

**对策**：写大文件用「临时文件名 → 写完再 `mv` 改名」的方式（原子落地），或直接 `rclone copy` 上传、不走挂载点。

### 6.2 缓存目录会占本地磁盘

`full` 模式下缓存默认落 `/tmp/rclone`（或 `~/.cache/rclone`）。**大文件会双份占盘**，且 `/tmp` 若被清理会中断上传。建议显式指定 `--cache-dir` 到持久目录 + `--vfs-cache-max-size` 封顶。

### 6.3 权限文件

`rclone.conf`（AK/SK）必须 `chmod 600`；挂载点若要多人用，配 `--allow-other` 并同步确认 `/etc/fuse.conf` 有 `user_allow_other`。

### 6.4 删除语义

FUSE 挂载点的删除通常进**桶的回收站/版本控制**（若桶开了版本）。误删可从对象存储侧恢复——这也是挂载方案的一个安全垫。

### 6.5 一致性

写入是**异步**的：`echo` 完立刻去云端看，可能还没有。验证一致性用 `rclone cat`/`rclone lsl` 对比，别只看本地 `ls`。

### 6.6 挂载点必须为空；未挂载时写入会「静默落本地」（实测踩到）

两个相邻的坑，凑一起非常坑：

1. rclone **拒绝挂到非空目录**：

   ```
   Fatal error: failed to mount FUSE fs: "/path" is not empty, use --allow-non-empty to mount anyway
   ```

   不要图省事加 `--allow-non-empty` —— 那会让目录里原有文件被挂载点**遮蔽**，看起来像「文件丢了」。

2. **服务没起来时往挂载点写文件，会直接写到本地目录**：返回成功、`ls` 也看得到 —— 但**云端什么都没有**，是彻头彻尾的假成功。

**所以验证顺序永远是**：先 `findmnt <挂载点>` 确认真的挂上了，**再**做写入测试；并且以云端（`rclone lsl`）为准，不看本地 `ls`。

## 七、运维速查

```bash
# 状态
systemctl status rclone-cos
journalctl -u rclone-cos -n 50 --no-pager      # 上传失败/重试都在这

# 手动同步（绕过挂载，最可靠）
rclone copy /local/path cos:<bucket>/remote/path -P
rclone sync /local/dir  cos:<bucket>/dir -P    # 注意：sync 会删目标端多余文件！

# 查看缓存占用
du -sh /tmp/rclone 2>/dev/null || du -sh ~/.cache/rclone
```

## 八、本机落地实例（Ubuntu 20.04 · 2026-09-27）

按 5.5 的用户级方案落地，全程未用 root，已实测验证：

| 项 | 值 |
|:---|:---|
| rclone | `v1.75.1`（官方二进制）→ `~/.local/bin/rclone` |
| 桶 | `<bucket>-<appid>`（`ap-shanghai`）|
| 挂载点 | `~/cos` |
| 缓存 | `~/.cache/rclone`，full 模式，上限 20G / 24h |
| 服务 | `rclone-cos.service`（**用户级**）· `enabled` + `active` |
| 额外依赖 | **`fuse3`**（Ubuntu 20.04 必须，见第三节的坑）|
| 密钥 | 子账号，仅该桶读写（最小权限）|

验证记录：

```bash
$ findmnt ~/cos
TARGET         SOURCE                  FSTYPE      OPTIONS
/home/<user>/cos cos:<bucket>-<appid> fuse.rclone rw,nosuid,nodev,relatime

# 写入 → 从对象存储侧确认（关键：以云端为准）
$ echo hello > ~/cos/test.txt && sleep 8
$ rclone lsl cos:<bucket>-<appid> --max-depth 1 | grep test
       32 2026-09-27 05:58:45.770711601 test.txt     # ✅ 云端已存在
```

> 桶内结构：`obsidian/`（笔记库，本笔记即存于此）、`backups/`、`.Trash-0/`。

## 相关文档

- `frp内网穿透部署实战.md` — 同样是「把远程资源接进本地」的隧道类方案，可对照理解 FUSE 挂载与端口转发的差异
- `DigitalGarden部署方案.md` — 本笔记库（Obsidian）即托管在此挂载点之上