---
{"dg-publish":true,"permalink":"/knowledge/language/python/01-Python文件操作/","title":"Python 文件操作","tags":["python","文件操作","IO","os模块"],"dg-note-properties":{"category":"Python 基础","title":"Python 文件操作","source":"csdn","created":"2021-07-05","updated":"2026-11-01","tags":["python","文件操作","IO","os模块"],"url":"https://blog.csdn.net/weixin_45536921/article/details/118492426"}}
---


# Python 文件操作

> 从「打开、读写、关闭」讲起，把 `with`、缓冲区、文件偏移量、`os` 模块一次讲清。
> 网络 IO 见 [[knowledge/language/python/03-Python网络编程UDP\|Python 网络编程 UDP]]。

## 一、为什么讲文件操作

几乎所有「读配置、处理日志、导数据、生成报表」的活儿，底层都是文件 IO。
Python 的文件操作 API 只有一组，但**踩坑点极多**：编码、换行符、缓冲区、指针位置、忘记关闭。

## 二、打开与关闭

### 2.1 `open()` 的基本形态

```python
f = open('data.txt', mode='r', encoding='utf-8')
content = f.read()
f.close()          # 必须手动关
```

| 参数 | 说明 |
|:----|:----|
| `mode` | 打开模式（见下表） |
| `encoding` | **中文场景必须显式写 `utf-8`**，否则用系统默认（Windows 是 GBK，必乱码） |
| `buffering` | 缓冲区大小；`-1` 默认，`0` 关闭（仅二进制） |
| `newline` | 换行符处理（读写 CSV 时常用 `newline=''`） |

### 2.2 打开模式一览

| 模式 | 含义 | 文件不存在 | 文件已存在 |
|:----|:----|:----|:----|
| `r` | 只读（默认） | 报错 `FileNotFoundError` | 从头读 |
| `w` | 只写 | 创建 | **清空内容**（危险） |
| `a` | 追加 | 创建 | 末尾追加 |
| `x` | 独占创建 | 创建 | 报错 `FileExistsError` |
| `r+` | 读写 | 报错 | 从头读写 |
| `w+` | 读写 | 创建 | **清空** |
| `a+` | 读写追加 | 创建 | 末尾追加 |

加 `b` 表示二进制：`rb` / `wb` / `ab`。**二进制模式不能指定 `encoding`。**

> `x` 模式是做「幂等初始化」的好帮手：只有第一次运行会创建文件，重复运行直接报错。

### 2.3 `with` 语句（**推荐永远用它**）

```python
with open('data.txt', 'r', encoding='utf-8') as f:
    content = f.read()
# 离开 with 块自动 close()，包括中途抛异常的情况
```

**为什么必须用 `with`**：手动 `close()` 在异常路径上会被跳过 → 文件句柄泄漏 → 达到系统上限后 `Too many open files`。
`with` 基于上下文管理器，`__exit__` 保证一定执行。

### 2.4 编码问题排查

```python
# 万能兜底写法
with open('a.txt', 'r', encoding='utf-8', errors='ignore') as f:
    ...
# errors: 'strict'(默认,报错) / 'ignore'(跳过) / 'replace'(替换成?) / 'backslashreplace'
```

Windows 上读到 GBK 文件：

```python
with open('gbk.txt', 'r', encoding='gbk') as f:   # 或 'gb18030'（超集，更保险）
    ...
```

## 三、读取的三种方式

```python
with open('data.txt', encoding='utf-8') as f:
    f.read()          # 一次读全部 → 返回 str（大文件会爆内存）
    f.read(1024)      # 读 1024 个字符
    f.readline()      # 读一行（含 \n）
    f.readlines()     # 所有行 → list[str]
```

**按行流式处理大文件（推荐）**——文件对象本身可迭代：

```python
with open('big.log', encoding='utf-8') as f:
    for line in f:                 # 惰性，一次只加载一行
        if 'ERROR' in line:
            print(line.rstrip())
```

> 常见错误：`f.readlines()` 读 10G 日志 → 内存打满。**优先 for line in f。**

## 四、写入

```python
with open('out.txt', 'w', encoding='utf-8') as f:
    f.write('hello\n')             # 返回写入字符数
    f.writelines(['a\n', 'b\n'])   # 注意：不会自动加换行
    print('c', file=f)             # print 重定向
```

**关键区别**：`write()` 要文件对象；`print(..., file=f)` 会自动加换行。

### 4.1 追加

```python
with open('app.log', 'a', encoding='utf-8') as f:
    f.write('2026-11-01 started\n')
```

## 五、缓冲区与 flush

Python 写入**先进缓冲区**，缓冲满或 close 时才真正落盘。所以：

```python
with open('a.txt', 'w') as f:
    f.write('something')
    f.flush()              # 强制刷盘（此时别的进程能看到）
```

**典型坑**：程序被 `kill -9` 或崩溃 → 缓冲区内容丢失。
日志类写入可加 `buffering=1`（行缓冲）或每次 `flush()`。

```python
open('a.txt', 'w', buffering=1)          # 文本模式：行缓冲
open('a.bin', 'wb', buffering=0)         # 二进制模式：不缓冲
```

## 六、文件偏移量（`tell` / `seek`）

```python
with open('a.txt', 'r+', encoding='utf-8') as f:
    print(f.tell())        # 0，当前指针位置
    f.read(3)
    print(f.tell())        # 3（文本模式下是「不透明数字」，不一定等于字节数）
    f.seek(0)              # 回到开头
    f.seek(0, 2)           # whence=2：移到末尾
```

| whence | 含义 |
|:----|:----|
| `0`（默认） | 从文件头偏移 |
| `1` | 从当前位置偏移 |
| `2` | 从文件末尾偏移 |

> **文本模式**下 `seek` 只能用在 `tell()` 返回过的位置，否则行为未定义；**二进制模式**下 `seek` 才是精确字节偏移。
> 取文件大小：`os.path.getsize(path)` 或 `f.seek(0, 2); size = f.tell()`。

## 七、`os` 与 `pathlib` 模块

### 7.1 `os` 常用

```python
import os

os.getcwd()                          # 当前目录
os.listdir('.')                      # 列出目录项
os.makedirs('a/b/c', exist_ok=True)  # 递归建目录（exist_ok 防重复报错）
os.remove('a.txt')                   # 删文件
os.rmdir('a')                        # 删空目录
os.rename('a', 'b')                  # 重命名/移动
os.path.exists('a.txt')
os.path.isfile('a.txt')
os.path.isdir('dir')
os.path.getsize('a.txt')
os.path.join('a', 'b', 'c.txt')      # 'a/b/c.txt'（跨平台）
os.path.basename('/a/b/c.txt')       # 'c.txt'
os.path.dirname('/a/b/c.txt')        # '/a/b'
os.path.splitext('a.tar.gz')         # ('a.tar', '.gz')
```

### 7.2 `pathlib`（现代写法，推荐新项目用）

```python
from pathlib import Path

p = Path('data') / 'a.txt'           # 用 / 拼路径，最直观
p.exists(); p.is_file(); p.suffix     # '.txt'
p.read_text(encoding='utf-8')
p.write_text('hi', encoding='utf-8')
p.parent.mkdir(parents=True, exist_ok=True)

for f in Path('logs').glob('*.log'):   # 通配
    print(f.name, f.stat().st_size)
for f in Path('.').rglob('*.py'):      # 递归 glob
    print(f)
```

**怎么选**：老代码/需要 `os.path` 兼容 → `os`；新项目 → `pathlib`。

## 八、常用场景代码

### 8.1 安全写入（先写临时文件再原子替换）

```python
import os, tempfile

def safe_write(path, text):
    d = os.path.dirname(os.path.abspath(path))
    fd, tmp = tempfile.mkstemp(dir=d)
    with os.fdopen(fd, 'w', encoding='utf-8') as f:
        f.write(text)
    os.replace(tmp, path)          # 原子替换
```

### 8.2 复制 / 移动 / 删除目录

```python
import shutil
shutil.copy('a.txt', 'b.txt')          # 复制文件
shutil.copytree('src', 'dst')          # 复制目录
shutil.move('a.txt', 'dir/')           # 移动
shutil.rmtree('dir')                   # 删目录（危险，不可恢复）
```

### 8.3 CSV / JSON 读写

```python
import csv, json

with open('a.csv', newline='', encoding='utf-8') as f:   # newline='' 防多余空行
    for row in csv.DictReader(f):
        print(row['name'])

with open('a.json', encoding='utf-8') as f:
    data = json.load(f)
with open('b.json', 'w', encoding='utf-8') as f:
    json.dump(data, f, ensure_ascii=False, indent=2)     # 中文不转义
```

## 九、常见坑速查

| 现象 | 原因 | 解法 |
|:----|:----|:----|
| `UnicodeDecodeError` | 编码不对 | 显式 `encoding='utf-8'` 或 `gb18030` |
| Windows 中文乱码 | 默认 GBK | 所有 open 都写 `encoding='utf-8'` |
| 写完文件内容是空的 | 缓冲未刷盘 / 程序崩溃 | `with` + `flush()` / `os.fsync` |
| `Too many open files` | 没 close | 全部改 `with` |
| 大文件内存暴涨 | `read()` / `readlines()` | `for line in f` 流式 |
| CSV 多出空行 | 未设 `newline=''` | `open(..., newline='')` |
| `FileNotFoundError`（写模式） | 父目录不存在 | 先 `mkdir(parents=True)` |
| 读到一半内容缺 | 文本模式 `seek` 乱用 | 用字节模式或 `readline` |

## 十、一句话总结

> **所有 open 都带 `encoding='utf-8'`，所有文件都用 `with`，大文件用 `for line in f`，新项目用 `pathlib`。**

## 相关文档

- [[knowledge/language/python/02-Python正则表达式\|Python 正则表达式]]
- [[knowledge/language/python/03-Python网络编程UDP\|Python 网络编程 UDP]]
