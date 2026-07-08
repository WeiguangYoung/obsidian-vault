---
{"dg-publish":true,"permalink":"/knowledge/Obsidian静态网站部署-DigitalGarden/","dg-note-properties":{}}
---


# Obsidian 静态网站部署 — Digital Garden

> 使用 Digital Garden 插件将 Obsidian 笔记发布为数字花园
> 基于 11ty (Eleventy)，零配置，GitHub Pages 自动部署

---

## 目录

1. [Digital Garden 简介](#1-digital-garden-简介)
2. [插件安装与配置](#2-插件安装与配置)
3. [GitHub 仓库与 Actions 自动构建](#3-github-仓库与-actions-自动构建)
4. [GitHub Pages 部署](#4-github-pages-部署)
5. [进阶配置](#5-进阶配置)
6. [注意事项](#6-注意事项)

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

仓库创建时会自带 Digital Garden 模板的 Actions 配置。如果用的是非 Fork 方式创建的仓库，在 `.github/workflows/` 下新建 `deploy.yml`：

```yaml
name: Deploy Digital Garden to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Build site
        run: npx @11ty/eleventy
        env:
          PAGES_REPO_NWO: ${{ github.repository }}

      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: _site

  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

---

## 4. GitHub Pages 部署

### 4.1 配置 Pages 部署源

1. 打开 GitHub 仓库 → **Settings → Pages**
2. 在 **Source** 中选择 **GitHub Actions**（而非默认的 Deploy from a branch）
3. 确认 `.github/workflows/deploy.yml` 已配置完成（见 3.2）

### 4.2 pathPrefix 配置（项目站点关键！）

> ⚠️ 如果你的站点部署在 `https://<user>.github.io/<repo>/` 这类**子路径**下，
> 必须给 11ty 设置 `pathPrefix`，否则所有内部链接会缺少仓库名称前缀导致 404。

在仓库根目录 `.eleventy.js` 的 `module.exports` 中增加 `pathPrefix`：

```js
module.exports = function(eleventyConfig) {
  // ... 原有配置 ...

  return {
    pathPrefix: "/你的仓库名/",   // ← 加上这一行
    dir: {
      input: "src/site",
      output: "dist",
      includes: "_includes",
      layouts: "_layouts"
    }
  };
};
```

> ✅ **用户根站点**（`https://<user>.github.io/`）无需此配置。

### 4.3 绑定自定义域名（可选）

1. 仓库 **Settings → Pages** → **Custom domain**
2. 输入你的域名（如 `notes.yourdomain.com`）
3. 在域名 DNS 管理中添加 CNAME 记录指向 `<user>.github.io`
4. 勾选 **Enforce HTTPS**（GitHub 会自动签发 SSL 证书）

### 4.4 发布流程一览

```
Obsidian 中写笔记
    ↓ 点击 🌱 发布
GitHub 仓库（保存笔记到 src/site/notes/）
    ↓ 自动触发
GitHub Actions（npm ci → npx @11ty/eleventy 构建）
    ↓
GitHub Pages（自动部署）
    ↓
🌐 公网访问
```

### 4.5 使用效果检查

部署完成后访问 `https://<user>.github.io/<repo>/`：
- ✅ 首页正常显示
- ✅ 笔记本页链接跳转正确
- ✅ 搜索、图谱功能正常
- ❌ 点击链接跳转 404 → 检查 pathPrefix 是否配置正确

---

## 5. 进阶配置

### 5.1 前端的暗黑/亮色主题切换

Digital Garden 默认使用暗黑主题。如需切换为亮色主题，编辑仓库中的 `src/site/styles/custom-style.scss`：

```scss
body {
    --background-primary: #ffffff;
    --background-secondary: #f5f5f5;
    --text-normal: #2c3e50;
    --text-accent: #2d7d46;
    /* 更多变量覆盖 */
}

h1, h2, h3, h4 {
    color: #2c3e50;
}

.theme-dark {
    display: none;
}
```

> 注意：
> - `custom-style.scss` 的变量会覆盖 `style.scss` 中的默认值
> - 修改后重新发布任意笔记触发 GitHub Actions 重新构建即可生效
> - 自定义样式只在已发布的笔记上生效

### 5.2 日记与反向链接

Digital Garden 原生支持：
- **日记/周记**自动归档
- **反向链接**页面显示
- **图谱视图**（通过 d3.js）

---

---

## 6. 注意事项

### 路径配置

1. ⚠️ **子路径部署必须配置 pathPrefix**：`https://<user>.github.io/<repo>/` 类型的站点务必在 `.eleventy.js` 配置 `pathPrefix: "/<repo>/"`，否则所有 wiki 链接解析为 `/notes/xxx/` 而非 `/<repo>/notes/xxx/`
2. ⚠️ **自定义域名**后 pathPrefix 应去掉或设置为 `"/"`，因为域名绑定的站点访问路径无仓库前缀

### 构建相关

3. ⚠️ **首次部署后**如果页面显示 404，检查 GitHub Actions 是否运行成功（仓库 Actions 标签页）
4. ⚠️ **npm ci 失败**时尝试改为 `npm install`，或检查 `package-lock.json` 是否存在
5. 📊 **构建时间**：53 篇笔记约需 1-2 分钟完成构建+部署

### Digital Garden 特有

6. **选择性发布是优势**：只有点 🌱 发布的笔记才会同步，无需担心整个 Vault 泄露
7. **保留仓库模板结构**：不要删除仓库中的 `src/site/` 等目录，插件需要完整的 11ty 模板结构

---

