---
{"dg-publish":true,"permalink":"/knowledge/Obsidian静态网站部署-DigitalGarden/","dg-note-properties":{}}
---


# Obsidian 静态网站部署 — Digital Garden

> 使用 Digital Garden 插件将 Obsidian 笔记发布为数字花园
> 基于 11ty (Eleventy)，零配置，Obsidian 插件直出

---

## 目录

1. [Digital Garden 简介](#1-digital-garden-简介)
2. [插件安装与配置](#2-插件安装与配置)
3. [GitHub 仓库与 Actions 自动构建](#3-github-仓库与-actions-自动构建)
4. [Cloudflare Pages 部署](#4-cloudflare-pages-部署)
5. [Cloudflare Access 登录认证](#5-cloudflare-access-登录认证)
6. [进阶配置](#6-进阶配置)
7. [注意事项](#7-注意事项)

---

## 1. Digital Garden 简介

[Digital Garden](https://dg-docs.ole.dev/) 是一个 Obsidian 插件，与 [Quartz](https://quartz.jzhao.xyz/) 类似，但最大的区别是：

| 特性 | Digital Garden | Quartz |
|------|---------------|--------|
| 操作方式 | **Obsidian 插件内操作** | 命令行操作 |
| 构建引擎 | 11ty (Eleventy) | Hugo |
| 笔记同步 | 插件内一键发布/更新 | git push 自动构建 |
| 需要本地终端 | ❌ 纯 Obsidian 内完成 | ✅ 需要终端操作 |
| 单篇/批量发布 | ✅ 支持单篇选择性发布 | 全量构建 |

**适合人群**：不想碰命令行、希望像发博客一样在 Obsidian 里点按钮就发布的用户。

---

## 2. 插件安装与配置

### 2.1 安装插件

1. 打开 Obsidian → **设置** → **社区插件** → **浏览**
2. 搜索 **Digital Garden**
3. 点击 **安装** → **启用**
4. 安装完成后左侧会出现一个 🌱 **花园** 图标

### 2.2 配置 GitHub 仓库

1. 在 GitHub 新建一个仓库（**公开**仓库 Free 计划免费；私有仓库需付费）
2. 回到 Obsidian → Digital Garden 插件设置
3. 填写以下配置：

| 配置项 | 值 |
|--------|-----|
| GitHub Repo | `用户名/仓库名` |
| Branch | `main` |
| Folder name | `src/site/notes/`（默认） |
| GitHub Token | 见下方步骤 |

### 2.3 生成 GitHub Token

1. 打开 [GitHub Settings → Developer settings → Personal access tokens → Fine-grained tokens](https://github.com/settings/tokens?type=beta)
2. **Generate new token**
   - Repository access: **Only select repositories** → 选你的花园仓库
   - Permissions → Contents: **Read and write**
3. 生成后复制 Token，粘贴到 Obsidian Digital Garden 设置的 `GitHub Token` 字段

### 2.4 发布你的第一篇笔记

1. 打开任意一篇 Obsidian 笔记
2. 点击右侧抽屉的 **🌱** 图标，或右键笔记 → **Digital Garden: Publish Note**
3. 插件会自动将笔记推送到你的 GitHub 仓库
4. 删除时：右键 → **Digital Garden: Unpublish Note**

---

## 3. GitHub 仓库与 Actions 自动构建

Digital Garden 插件会把发布的笔记 push 到 GitHub 仓库。要实现自动构建部署，需要配置 GitHub Actions。

### 3.1 仓库目录结构

Digital Garden 插件推送到 GitHub 后的仓库结构：

```
你的花园仓库/
├── .github/
│   └── workflows/
│       └── deploy.yml        # 新建
├── src/
│   └── site/
│       ├── notes/             # 发布的笔记（插件自动写入）
│       ├── _data/
│       ├── _includes/
│       ├── css/
│       └── ...
├── .eleventy.js               # 11ty 配置文件
├── package.json
├── _config.yml
└── README.md
```

### 3.2 GitHub Actions 工作流

在仓库创建 `.github/workflows/deploy.yml`：

```yaml
name: Deploy Digital Garden to Cloudflare Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Build site
        run: npx @11ty/eleventy

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: _site
```

> 💡 Digital Garden 的 11ty 构建输出目录默认为 `_site`，与 Cloudflare Pages 默认配置一致。
> 如果直接使用 Cloudflare Pages 连接仓库，Cloudflare 会自动检测并可以使用默认设置，无需此 Actions 文件。

---

## 4. Cloudflare Pages 部署

### 4.1 创建 Pages 项目

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. 进入 **Workers & Pages → Pages**
3. 点击 **Create a project → Connect to Git**
4. 授权 GitHub 并选择你的 Digital Garden 仓库

### 4.2 构建配置

| 配置项 | 值 |
|--------|-----|
| Framework preset | **Eleventy**（自动检测）或 None |
| Build command | `npx @11ty/eleventy` |
| Build output directory | `_site` |
| Node.js version | 20+ |

### 4.3 绑定自定义域名（可选）

1. Pages 项目 → **Custom domains** → **Set up a custom domain**
2. 输入你的域名（如 `garden.yourdomain.com`）
3. Cloudflare 自动处理 DNS 和 SSL 证书

### 4.4 发布流程一览

```
Obsidian 中写笔记
    ↓ 点击 🌱 发布
GitHub 仓库（保存笔记）
    ↓ 自动触发
Cloudflare Pages（构建 + 部署）
    ↓
🌐 公网访问
```

---

## 5. Cloudflare Access 登录认证

> 网关级认证，**零代码修改**，未通过验证的请求在到达网站前就被拦截。

### 5.1 前置准备

1. Cloudflare 账号（免费即可）
2. 站点已部署到 Cloudflare Pages
3. 已绑定自定义域名（推荐）

### 5.2 接入 Cloudflare Zero Trust

1. 登录 [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)
2. 选择 **Free 计划**（50 用户/月免费额度），填写团队名称
3. 左侧菜单 **Access → Applications → Add an application**
4. 选择 **Self-hosted** 类型，点击 Next

### 5.3 配置应用与认证策略

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| Application name | My Digital Garden | 后台标识，可自定义 |
| Session Duration | 24 hours | 认证有效期 |
| Application domain | garden.yourdomain.com | 站点域名（须与 Pages 绑定一致） |

**Policy 配置（至少一条 Allow 策略）：**

- **个人使用**：`Action = Allow`，`Include = Emails → 你的邮箱`
- **团队共享**：`Action = Allow`，`Include = Emails ending in → @company.com`

💡 Free 计划支持 Email OTP、GitHub、Google、Microsoft 等多种免密登录。

### 5.4 绑定 Cloudflare Pages 项目

1. 回到 Cloudflare 主控台 → **Workers & Pages → 你的 Pages 项目**
2. **Settings → Access** 标签页
3. 点击 **Add Access policy**，选择刚创建的 Application
4. 保存后自动为该 Pages 项目启用边缘认证

### 5.5 验证与测试

1. 浏览器无痕模式访问站点域名
2. ✅ 应看到 Cloudflare 认证页面（而非笔记内容）
3. 输入授权邮箱 → 收到验证码 → 完成验证
4. ✅ 24h 内无需重新登录
5. ✅ **安全检查**：未登录状态下直接访问页面源码 → 返回的是认证页 HTML

---

## 6. 进阶配置

### 6.1 自定义主题

Digital Garden 支持主题切换，在 Obsidian 插件设置中可选：
- **Default**：简洁白色主题
- **Dark**：深色模式
- **Custom CSS**：在仓库 `src/site/css/` 下自定义样式

### 6.2 日记与反向链接

Digital Garden 原生支持：
- **日记/周记**自动归档
- **反向链接**页面显示
- **图谱视图**（通过 d3.js）

### 6.3 自定义登录页外观

**Zero Trust → Access → Applications → 编辑应用 → Appearance**

- 标题、副标题、Logo
- 背景颜色、按钮样式
- 多语言支持（含中文）

### 6.4 API/自动化访问

若需通过脚本抓取笔记内容：

1. 在 Access Policy 中添加 **Service Token** 认证
2. 脚本请求时携带 Header：`CF-Access-Client-Id` + `CF-Access-Client-Secret`

---

## 7. 注意事项

### 认证相关

1. ⚠️ **不要混用客户端鉴权**：Cloudflare Access 是网络层拦截，勿在 Digital Garden 代码中加 JS 登录检查
2. ⚠️ **缓存问题**：开启认证后仍能直接访问？检查 Pages 缓存设置，或在 **Rules → Cache Rules** 中禁用缓存
3. 📊 **免费额度上限**：Free 计划限制 **50 个独立用户/月**
4. 🔒 **敏感数据兜底**：极度敏感的笔记建议用 `obsidian-encrypt` 插件端到端加密

### Digital Garden 特有

5. **仓库需公开**：Cloudflare Free 计划只支持公开仓库。如需私有仓库，需 Cloudflare Teams 付费计划或 Vercel Pro
6. **选择性发布是优势**：只有你点 🌱 发布的笔记才会同步，无需担心整个 Vault 泄露
7. **不要删除仓库中的 `.github/`**：Digital Garden 插件需要 `src/site/` 目录结构完整才能正常构建

---

## 📊 Quartz vs Digital Garden 决策参考

| 场景 | 推荐 | 原因 |
|------|------|------|
| 不想碰命令行 | **Digital Garden** | Obsidian 插件内操作，点按钮发布 |
| 笔记量大（100+） | **Quartz** | 全量构建更高效 |
| 选择性发布 | **Digital Garden** | 单篇/批量可控 |
| 全量发布 | **Quartz** | 软链接一次配置，推就完事 |
| 追求自由度/自定义 | **Quartz** | 配置更灵活，主题丰富 |
| 图省心 | **Digital Garden** | 开箱即用 |

---

> 📌 最后更新：2026-07-07
