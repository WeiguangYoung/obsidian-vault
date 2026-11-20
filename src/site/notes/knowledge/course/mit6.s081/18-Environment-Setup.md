---
{"dg-publish":true,"permalink":"/knowledge/course/mit6.s081/18-Environment-Setup/","title":"6.S081 · 环境配置","tags":["MIT6.1810","xv6","操作系统","环境配置"],"dg-note-properties":{"title":"6.S081 · 环境配置","tags":["MIT6.1810","xv6","操作系统","环境配置"],"date":"2026-11-18"}}
---


# 6.S081 · 环境配置

> 从零搭起 xv6-riscv 的开发环境：Ubuntu + 工具链 + QEMU

返回 [[knowledge/course/mit6.s081/index\|📚 6.S081 目录]] · [[knowledge/course/index\|📚 课程资源]] · [[首页\|🏠 首页]]

---

## 一、基础系统

| 步骤 | 说明 |
|:--|:--|
| 1 | 准备 Ubuntu（20.04 LTS 最省事，工具链兼容性最好） |
| 2 | 安装 VS Code —— 主要用来读 `kernel/` 源码、跨文件跳转 |
| 3 | 安装 git —— 切换 Lab 分支必需 |
| 4 | 安装 `gparted` —— 虚拟机磁盘扩容时用得上 |

## 二、RISC-V 工具链

官方工具链安装说明在 MIT 的 tools 页面。**最大的坑在子模块**：`riscv-gnu-toolchain` 的 git 子模块极多，其中 `qemu` 子模块体积大、经常下载中断。

处理顺序建议：

1. 先按官方页面走一遍 `riscv-gnu-toolchain` 的完整安装（`--prefix` 指到自己有写权限的目录）
2. 如果卡在子模块，改用国内镜像拉 `qemu` 源码
3. 依赖包缺失时，按报错逐个补 `apt install`

## 三、磁盘空间

工具链 + QEMU 源码 + 编译产物会吃掉十几 GB，虚拟磁盘容易不够。先查清现状：

```shell
df -h          # 看各分区剩余
du -sh *       # 看当前目录谁在占空间
```

空间不足时优先扩虚拟磁盘，而不是删工具链——重装一次要等很久。

## 四、安装后的验证

```shell
riscv64-unknown-elf-gcc --version     # 交叉编译器
qemu-system-riscv64 --version         # 模拟器
```

两条都能输出版本号，说明工具链就绪。之后在 xv6 源码目录：

```shell
make qemu
```

看到 `xv6 kernel is booting` 并进入 `$` 提示符，环境就通了。

## 五、调试环境

配合 GDB 的用法见 [[knowledge/course/mit6.s081/19-GDB-Debug\|19-GDB-Debug]]。启动调试模式的命令：

```shell
make CPUS=1 qemu-gdb
```

单核（`CPUS=1`）能让 GDB 单步时的输出干净很多，初学强烈建议。

---

## 参考链接

| 资源 | 链接 |
|:--|:--|
| 官方工具链安装说明 | https://pdos.csail.mit.edu/6.S081/2020/tools.html |
| QEMU 清华镜像 | https://mirrors.tuna.tsinghua.edu.cn/help/qemu.git/ |
| Ubuntu 镜像源配置 | https://developer.aliyun.com/article/639051 |