---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c-labs/Lab00/","title":"CS61C FA26 · Lab 00 · 环境配置与 SSH","tags":["CS61C","lab","lab00","SSH","Vim","Git","Python"],"dg-note-properties":{"title":"CS61C FA26 · Lab 00 · 环境配置与 SSH","tags":["CS61C","lab","lab00","SSH","Vim","Git","Python"],"date":"2026-11-20","source":"https://cs61c.org/fa26/labs/lab00/"}}
---


# Lab 00 · Intro, Setup

> UC Berkeley · Great Ideas in Computer Architecture · Fall 2026
> 代码仓库：`github.com/61c-teach/fa26-lab-starter`
> 本地路径：`/home/yang/.openclaw/workspace/fa26-lab-starter/lab00/`

返回 [[knowledge/course/cs61c-labs/index\|🧪 Labs 总览]] · [[knowledge/course/cs61c/index\|📚 CS61C 目录]]

---

## 这一课在做什么

Lab 0 是**纯配置课**：不写算法，只把后面一学期要用的所有服务、工具、机器全部打通，最后用几个小脚本验证环境。

课程官方明说：**这 lab 比较长，但请仔细读**，因为它覆盖了后续所有作业的前置条件。也正因为不涉及知识点，Ed 上的提问规则放宽了——**可以发私有的问题**（这通常是违规的）。

> ⚠️ 官方还提供了 [common errors 页面](https://cs61c.org/fa26/resources/common-errors/)：去 Ed 提问或去 OH 之前请先自查，能省下大量等待时间。

整体路线：

```text
账号与服务 → 本机装软件 → 接 VPN → 拿 instructional 账号
    → SSH 上 hive 机器 → 配 ~/.ssh/config 快捷键
    → 验证 Python 行为（code.py）→ 收集 debug 信息 → Git 初始化
```

---

## 文件清单

| 文件 | 状态 | 说明 |
|---|---|---|
| `code.py` | ⬜ 待修改 | Python 行为演示；gutter 级别的“改一行”练习 |
| `gen-debug.sh` | ✅ 已完整 | 把 OS/shell/git/python/java 版本写入 `debug.txt` |
| `get-ssh-key.sh` | ✅ 已完整 | 检测已有 SSH key，没有就生成 ed25519 |
| `init.sh` | ✅ 已完整 | Hive 上给 git 仓库打 tag 并重置到 lab00 起点 |
| `debug.txt` | ⬜ 产物 | 由 `gen-debug.sh` 生成，需提交 |

> `code.py` / `gen-debug.sh` / `get-ssh-key.sh` 在**本地机器**运行；`init.sh` 是**在 hive 上**执行的仓库初始化脚本，按官方说明用它把仓库重置到 lab00 起点。

---

## Exercise 1 · Accessing Services

**目标**：确认下列所有服务都能登录，缺一个下面几步都会卡住。

| 服务 | 作用 | 检查点 |
|---|---|---|
| **Ed** | 主讨论区 / 提问 | 已加入 61C Ed course |
| **Gradescope** | 交作业、看分 | 已加入 61C Gradescope course |
| **PrairieLearn** | 在线练习题 | `Add or remove courses` → 选 CS61C Fall 2026 |
| **OH Queue** | 排队上 office hours | 能登录 `oh.cs61c.org` |
| **YouTube** | 录播视频 | 能播放私享测试视频 |
| **Zoom** | Live lecture / OH | `Sign In with SSO` → 域填 `berkeley` |

**Zoom 的坑**：SSO 登录时输入的是 `berkeley`，最终域名会是 `berkeley.zoom.us`，不要手抖输成 `berkeley.edu`。

### Gradar 绑定

Gradar 是查分的网页端，需要把学校账号和它绑定：

```text
① 打开 Gradar，登录
② 确认课程显示为 CS 61C Fall 2026
③ 点 Profile → Link Account → 登录
```

### 校园 VPN（前置知识）

有些地区/网络会屏蔽课程服务。如果访问不了任何资源，需要装 **Berkeley bSecure Remote Access VPN**（本 lab 后面会装）。注意：装好后**整个 lab 剩余部分都要保持连接**。

---

## Exercise 2 · Installing Software

**目标**：本机具备 `git`、`curl`、JDK 17、`python3`。按自己的 OS 走，别跨平台抄命令。

### Ubuntu Linux

Ubuntu 18.04+ 官方源里就有全部依赖，一条命令搞定：

```bash
sudo apt update
sudo apt install curl git openjdk-17-jre openssh-client python3 python3-pip
```

终端 + 任意现代浏览器即可，不需要猜别的。

### macOS

```bash
xcode-select --install    # Xcode 命令行工具（提供 gcc 的 clang 版本）
java -version             # 需要 Java 16+，否则装 Adoptium OpenJDK 17
python3 --version         # 需要 Python 3.6+
```

**SSL 证书自检**（macOS 上 Python 经常出问题）：

```bash
python3 -c 'import urllib.request; urllib.request.urlopen("https://inst.eecs.berkeley.edu")'
```

- **没有输出** = 正常
- 报 `certificate verify failed: unable to get local issuer certificate` = 证书没装：
  1. Finder 打开 `/Applications`
  2. 进入 `Python 3.11`（版本号按你的来）
  3. 双击运行 `Install Certificates.command`
  4. **关掉所有终端窗口**再新开一个
  5. 重新跑上面那行自检

> 为什么必须重开终端：`Install Certificates.command` 改的是 Python 安装目录里的证书包，已启动的进程缓存了旧的环境。

### Windows

1. 安装 **Git Bash**（git-scm.com）。安装器的终端选项两种都行：
   - `Use Windows' default console window`
   - `Use MinTTY`（默认，体验更好）
2. **必须用 Git Bash**，不要用 `cmd.exe`——下面的 shell 脚本和路径斜杠在 cmd 里不成立。
3. `java -version` → 需要 16+，否则装 Adoptium OpenJDK **17 或 21**
4. `python3 --version` → 需要 3.6+

### 其他系统

| 发行版 | 支持程度 |
|---|---|
| Debian / Fedora / Arch / Alpine / NixOS 等 | 依赖装齐大概率能跑，但**官方只测 Ubuntu** |
| *BSD / HaikuOS / TempleOS / 其他 | **不提供支持**，请改用教学机 |

---

## Exercise 3 · Instructional Accounts & Hive Machines

**目标**：拿到 `cs61c-???` 账号，能 SSH 登进 hive 机器，并把提示符调成课程要求的样式。

### 先搞清两个概念

- **hive 机器** = 教学服务器（`hive1` ~ `hiveN`），所有 lab/项目都在上面跑。
- [Hivemind](https://hivemind.eecs.berkeley.edu) 可以查每台机器忙不忙。
  > ⚠️ Hivemind 里也列出 `ashby`、`cedar` 这类**不以 hive 开头**的机器——**它们不能用于 61C**。

### 网络前提

| 网络 | 能否 SSH |
|---|---|
| 校园 CalVisitor WiFi | ❌ **封 SSH** |
| 家宽 / 蜂窝 | 视地区而定，可能被墙 |
| 校园 bSecure VPN | ✅ 推荐 |

> **常见错误第一名的原因**就是人在 CalVisitor 上。

### 步骤

**① 注册 instructional 账号**

去 WebAccount（`acropolis.cs.berkeley.edu/~account/webacct/`）申请 `cs61c` 账号。

- 密码后面一直要用，记好。
- 选上课/在 waitlist 后**可能等 1~2 天**才能创建。
- 并发注册 / 填了 "Plan to Enroll Later" 的同学，提交后 5 天内可创建。

**② SSH 登录**

```bash
ssh cs61c-???@hive2.cs.berkeley.edu
```

- `cs61c-???` 换成你的用户名。
- **输密码时屏幕不会有任何回显**，这是正常现象，盲打回车即可。

报错对照：

| 报错 | 含义 | 处理 |
|---|---|---|
| `Permission denied, please try again` | 密码错 / 账号还没生效 | 核对密码；等一两天再试 |
| `Connection timeout` | 网络不通（多半在 CalVisitor 或被墙） | 换网络 / 开 VPN |
| `Connection refused` | 这台机器挂了 | **换 `hive3` 等其他机器** |

**③ 首次登录填写信息**

可能被要求填名字和邮箱：

- 邮箱填 **Berkeley 邮箱**
- ⚠️ **仔细核对邮箱、SID、姓名拼写**——写错了要去 OH 改

**④ 校验**

```bash
check-register     # 检查信息是否正确
re-register        # 不对就重新登记
```

**⑤ 检查提示符样式**

正确的样子是**红黄色**提示符。如果是白色，说明 dotfiles 没生效：

```bash
/home/ff/cs61c/bin/fix-dotfiles && exit
```

`exit` 之后再从头执行第 ② 步重新登录。

**⑥ 退出**

```bash
exit
```

### 配 SSH 别名（顺便学 Vim）

配好后可以用 `ssh hive2` 代替一长串 `ssh cs61c-???@hive2.cs.berkeley.edu`。

```bash
mkdir -p ~/.ssh        # 确保目录存在
vim ~/.ssh/config      # 打开（或新建）配置文件
```

**Vim 最小操作集**：

| 操作 | 按键 |
|---|---|
| 进入插入模式 | `i`（左下角出现 `-- INSERT --`） |
| 退出插入模式 | `Esc` |
| 保存并退出 | `:wq` + 回车 |
| 不保存退出 | `:q!` + 回车 |

**要写的内容**（`Host` 行 + 缩进的 `HostName` / `User`）：

```txt
Host hive2
    HostName hive2.cs.berkeley.edu
    User cs61c-???
```

> ⚠️ 如果你以前上过 **CS61B**，`~/.ssh/config` 里可能残留这两行，会**覆盖**你的新配置：
>
> ```txt
> Host *.cs.berkeley.edu *.eecs.berkeley.edu
> IdentityFile ~/.ssh/cs61b_id_rsa
> ```
>
> 用不到的请删掉，否则 `ssh hive2` 会去用 61B 的 key 导致认证失败。

> ⚠️ **待补充**：官方 md 在这一步之后被截断了（原文 271 行到此结束）。Lab 0 后面的 Exercise（Python 基础、`code.py`、`gen-debug.sh`、`get-ssh-key.sh`、`init.sh`、Git 配置等）**请对照网页原文补齐**。

---

## 骨架脚本逐个看

虽然正文缺失，但这几个脚本能反推出它们的作用，先弄清楚再跑。

### `gen-debug.sh` — 收集环境信息

```bash
#!/bin/bash

if [[ "$HOSTNAME" == hive* ]]; then
  echo "Please run this on your local machine, not a hive machine."
fi

echo "OSTYPE='$OSTYPE'" > debug.txt
echo "BASH_VERSION='$(bash --version 2>&1 | cat)'" >> debug.txt
echo "SH_VERSION='$(sh --version 2>&1 | cat)'" >> debug.txt
echo "GIT_VERSION='$(git --version 2>&1 | cat)'" >> debug.txt
echo "PYTHON3_VERSION='$(python3 --version 2>&1 | cat)'" >> debug.txt
echo "PYTHON_VERSION='$(python --version 2>&1 | cat)'" >> debug.txt
echo "PY_VERSION='$(py --version 2>&1 | cat)'" >> debug.txt
echo "JAVA_VERSION='$(java -version 2>&1 | cat)'" >> debug.txt
```

要点：

- `> debug.txt` 第一次**覆盖**，后面 `>>` **追加**——最终是一个文件。
- `2>&1` 把 stderr 并到 stdout：`java -version` 的版本信息是打到 **stderr** 的，不加就抓不到。
- 注意 `if` 只打印警告，**并没有退出**——在 hive 上跑仍会生成文件，只是内容没意义。

```bash
chmod +x gen-debug.sh   # 若已可执行可跳过
./gen-debug.sh
cat debug.txt
```

> ⚠️ 脚本假设当前工作目录可写，因为 `debug.txt` 是相对路径。

### `get-ssh-key.sh` — 检测/生成 SSH key

```bash
#!/usr/bin/env bash
set -euf -o pipefail

SSH_HOST=${1:-hive4.cs.berkeley.edu}
```

| 选项 | 含义 |
|---|---|
| `-e` | 任一命令失败即退出 |
| `-u` | 用到未定义变量即报错 |
| `-f` | 禁用通配符展开 |
| `-o pipefail` | 管道中任一环节失败，整条管道算失败 |

`${1:-默认值}` 让脚本支持传参：`./get-ssh-key.sh hive2.cs.berkeley.edu`。

核心逻辑——**先问 ssh 自己会用哪些 key**，再逐个检查是否真的存在：

```bash
for OUTPUT in $(ssh "$SSH_HOST" -G | grep "^identityfile " | cut -d " " -f2-)
do
    filename="${OUTPUT/#\~/$HOME}"      # bash 替换：开头的 ~ 换成 $HOME
    if [[ -f $filename ]]; then
        echo "You already have an SSH key at: $OUTPUT"
        exit 0                           # 已有 → 直接退出
    fi
done

echo "I couldn't find an existing SSH key for you, so let's make one now..."
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ''
```

- `ssh -G` 打印**解析后的完整配置**，`grep identityfile` 捞出所有候选 key 路径。
- `${OUTPUT/#\~/$HOME}` 是 bash 的**前缀替换**语法（`#` 表示只替换开头）。
- `-t ed25519`：现代推荐的曲线算法，比 RSA 短且快。
- `-N ''`：空 passphrase，方便自动化；**代价是私钥没有密码保护**。

> ⚠️ **别把私钥 `~/.ssh/id_ed25519` 提交到 git**！要上传的是公钥 `.pub`。

### `init.sh` — 仓库初始化（在 hive 上执行）

```bash
#!/bin/bash
set -eu

lab00_ref="starter/lab00-oski"

git fetch starter

lab00_start_ref="$(git rev-parse "${lab00_ref}~1")"

git tag -f lab00-backup
git push -f origin refs/tags/lab00-backup
git reset --hard "$lab00_start_ref"
git push -f origin "${lab00_ref}:main"
git update-ref refs/remotes/origin/main "$lab00_start_ref" "$lab00_ref"
```

逐行拆解：

| 行 | 作用 |
|---|---|
| `lab00_ref="starter/lab00-oski"` | 起点分支名（`~1` 表示它的前一个提交） |
| `git fetch starter` | 从 starter remote 拉取，但**不改工作区** |
| `git rev-parse "${lab00_ref}~1"` | 取父提交的 SHA，作为真正的起点 |
| `git tag -f lab00-backup` | 给当前 HEAD（你的旧内容）打标签，作为备份 |
| `git push -f origin refs/tags/lab00-backup` | 备份推到自己的 origin |
| `git reset --hard "$lab00_start_ref"` | **丢弃本地改动**，工作区回到起点 |
| `git push -f origin "${lab00_ref}:main"` | 把起点推到 origin 的 `main` |
| `git update-ref ...` | 同步本地 remote-tracking ref，避免 status 显示错乱 |

> ⚠️ `reset --hard` + `push -f` 是**破坏性操作**。脚本的设计意图是：先把你的东西推到 `lab00-backup` tag 里，再推平 `main`。所以**跑之前确认 `git status` 干净**，或者自己先 `git stash`。

---

## `code.py` — Python 行为观察

```python
def get_airspeed_velocity_of(unladen_swallow):
  if unladen_swallow.type == "african":
    return # redacted
  elif unladen_swallow.type == "european":
    return # redacted

# pretend there's code here...

def fizzbuzz(num):
  if num == 3: # edit this line
    print(f"{num}: fizz")
  if num == 5: # edit this line
    print(f"{num}: buzz")

for i in range(1, 20):
  fizzbuzz(i)
```

按骨架里的 `# edit this line` 注释，把**相等判断**改成**整除判断**才是真正意图的 FizzBuzz：

```python
def fizzbuzz(num):
  if num % 3 == 0:   # 改：3 的倍数
    print(f"{num}: fizz")
  if num % 5 == 0:   # 改：5 的倍数
    print(f"{num}: buzz")
```

注意这里**没有 `elif`**，所以 15 会同时打印 `fizz` 和 `buzz`。

运行：

```bash
python3 code.py
```

> ⚠️ **待补充**：Exercise 对 `code.py` 的具体问题（例如“为什么不写 `%` 时输出不对”“`return` 后面什么都没有会返回什么”）在截断的正文里，请按网页原文核对。

---

## 易错点汇总

| 现象 | 原因 | 解法 |
|---|---|---|
| SSH 连接超时 | 在 CalVisitor WiFi | 换网络 / 开 bSecure VPN |
| 密码怎么打都没反应 | 正常，终端不回显 | 盲打后回车 |
| `Permission denied` | 账号未生效 | 等 1~2 天，或 `re-register` |
| 提示符白色 | dotfiles 未加载 | `/home/ff/cs61c/bin/fix-dotfiles && exit` |
| `ssh hive2` 认证失败 | 61B 的 `Host *.cs.berkeley.edu` 覆盖了配置 | 删掉那两行 |
| `debug.txt` 里 Java 版本为空 | 没把 stderr 重定向 | 脚本已用 `2>&1`，自行手敲时别漏 |
| macOS Python 证书报错 | 未安装根证书 | 跑 `Install Certificates.command` 后**重开终端** |
| Windows 命令全报错 | 用了 cmd.exe | 改用 **Git Bash** |

---

## 相关

- Labs 总览：[[knowledge/course/cs61c-labs/index\|🧪 Labs]]
- 后续：[[knowledge/course/cs61c-labs/Lab01\|Lab 01 · C 基础]]
- 对应讲次：[[knowledge/course/cs61c/01-Course-Introduction\|L01 课程介绍]] · [[knowledge/course/cs61c/02-Number-Representation\|L02 数字表示]]
