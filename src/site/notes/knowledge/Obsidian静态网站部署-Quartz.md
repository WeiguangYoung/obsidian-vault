---
{"dg-publish":true,"permalink":"/knowledge/Obsidian静态网站部署-Quartz/","dg-note-properties":{}}
---


# Obsidian 静态网站部署 — Quartz

> 将 Obsidian 笔记发布为静态网站，使用 Quartz 框架 + Cloudflare Pages 部署 + Cloudflare Access 认证保护

---

## 目录

1. [Quartz 简介](#1-quartz-简介)
2. [本地初始化与配置](#2-本地初始化与配置)
3. [GitHub 仓库与 Actions 自动构建](#3-github-仓库与-actions-自动构建)
4. [Cloudflare Pages 部署](#4-cloudflare-pages-部署)
5. [Cloudflare Access 登录认证](#5-cloudflare-access-登录认证)
6. [从其他平台迁移](#6-从其他平台迁移)
7. [进阶配置](#7-进阶配置)
8. [注意事项](#8-注意事项)

---

## 1. Quartz 简介

[Quartz](https://quartz.jzhao.xyz/) 是一个基于 Hugo 的静态站点生成器，专门为 Obsidian 笔记库设计：

- **零配置发布**：直接从 Obsidian Vault 构建
- **双向链接 & 图谱**：原生支持 `[[wikilink]]` 和 Graph View
- **全文本搜索**：内置 Fuse.js 模糊搜索
- **多种主题**：支持自定义 CSS 和主题切换
- **响应式设计**：桌面/移动端自适应

替代方案：[Digital Garden](https://dg-docs.ole.dev/)（基于 11ty）、[Obsidian Publish](https://obsidian.md/publish)（官方付费服务）

---

## 2. 本地初始化与配置

### 2.1 安装依赖

```bash
# 需要 Node.js >= 18
node -v

# 克隆 Quartz 模板
git clone https://github.com/jackyzha0/quartz.git my-digital-garden
cd my-digital-garden

# 安装依赖
npm install
```

### 2.2 配置内容源

Quartz 默认将 `content/` 目录作为笔记源。有两种方式：

**方式 A（推荐——软链接）：**

```bash
# 创建指向 Obsidian Vault 的软链接
rm -rf content
ln -s /path/to/your/obsidian/vault content
```

**方式 B（复制——适合选择性发布）：**

```bash
# 手动将需要发布的笔记复制到 content/ 目录
cp -r /path/to/vault/selected-folder content/
```

### 2.3 核心配置

编辑 `quartz.config.ts`：

```typescript
// 站点基本信息
const config: QuartzConfig = {
  configuration: {
    pageTitle: "🌱 我的数字花园",
    enableSPA: true,          // 单页应用模式
    enablePopovers: true,     // 悬停预览
    analytics: null,          // 可选：plausible / google / umami
    locale: "zh-CN",          // 中文界面
    baseUrl: "your-domain.com",  // 部署域名
    ignorePatterns: [         // 忽略的文件夹/文件
      "templates",
      "私密笔记",
      "**/*.excalidraw.md"
    ],
    defaultDateType: "modified",  // 默认显示修改日期
    theme: {
      cdnCaching: true,
      typography: {
        header: "Noto Serif SC",
        body: "Noto Sans SC",
        code: "Fira Code"
      },
    },
  },
  plugins: {
    // 按需启用/禁用插件
    transformers: [...],
    filters: [...],
    emitters: [...],
  },
};
```

### 2.4 本地预览

```bash
npx quartz build --serve
# 访问 http://localhost:8080
```

---

## 3. GitHub 仓库与 Actions 自动构建

### 3.1 推送到 GitHub

```bash
# 在 GitHub 创建新仓库，然后：
git remote add origin https://github.com/你的用户名/你的仓库.git
git push -u origin main
```

### 3.2 GitHub Actions 工作流

在仓库创建 `.github/workflows/deploy.yml`：

```yaml
name: Deploy Quartz to Cloudflare Pages

on:
  push:
    branches: [main]
  workflow_dispatch:  # 手动触发

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Build Quartz
        run: npx quartz build

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: public
```

> 💡 如果直接使用 Cloudflare Pages 连接 GitHub 仓库，Cloudflare 会自动检测构建配置，可以不使用此 Actions 文件。

---

## 4. Cloudflare Pages 部署

### 4.1 创建 Pages 项目

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. 进入 **Workers & Pages → Pages**
3. 点击 **Create a project → Connect to Git**
4. 授权 GitHub 并选择你的 Quartz 仓库

### 4.2 构建配置

| 配置项 | 值 |
|--------|-----|
| Framework preset | None |
| Build command | `npx quartz build` |
| Build output directory | `public` |
| Node.js version | 20+ |

### 4.3 绑定自定义域名

1. Pages 项目 → **Custom domains** → **Set up a custom domain**
2. 输入你的域名（如 `garden.yourdomain.com`）
3. Cloudflare 自动处理 DNS 和 SSL 证书
4. 完成后更新 `quartz.config.ts` 中的 `baseUrl`

---

## 5. Cloudflare Access 登录认证

> 网关级认证，未通过验证的请求在到达网站之前就被拦截，**零代码修改**。

### 5.1 前置准备

1. Cloudflare 账号（免费即可）
2. 站点已部署到 Cloudflare Pages
3. 已绑定自定义域名（推荐）

### 5.2 第一步：接入 Cloudflare Zero Trust

1. 登录 [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)
2. 选择 **Free 计划**（50 用户/月免费额度），填写团队名称
3. 左侧菜单 **Access → Applications → Add an application**
4. 选择 **Self-hosted** 类型，点击 Next

### 5.3 第二步：配置应用与认证策略

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| Application name | My Obsidian Vault | 后台标识，可自定义 |
| Session Duration | 24 hours | 认证有效期 |
| Application domain | vault.yourdomain.com | 站点域名（须与 Pages 绑定一致） |

**Policy 配置（至少一条 Allow 策略）：**

- **个人使用**：`Action = Allow`，`Include = Emails → 你的邮箱`
- **团队共享**：`Action = Allow`，`Include = Emails ending in → @company.com`
- **临时分享**：`Action = Allow`，`Include = Single-use PIN` → 生成一次性验证码

💡 Free 计划支持 Email OTP、GitHub、Google、Microsoft 等多种免密登录方式，无需自建用户系统。

### 5.4 第三步：绑定 Cloudflare Pages 项目

1. 回到 Cloudflare 主控台 → **Workers & Pages → 你的 Pages 项目**
2. **Settings → Access** 标签页
3. 点击 **Add Access policy**，选择刚创建的 Application
4. 保存后自动为该 Pages 项目启用边缘认证

### 5.5 第四步：验证与测试

1. 浏览器无痕模式访问站点域名
2. ✅ 应看到 Cloudflare 认证页面（而非笔记内容）
3. 输入授权邮箱 → 收到验证码 → 完成验证
4. ✅ 验证通过后自动跳转至笔记站点，24h 内无需重新登录
5. ✅ **安全检查**：未登录状态下直接访问 `/content/sensitive-note.md` 或查看页面源码 → 返回认证页 HTML，而非笔记内容

---

## 6. 从其他平台迁移

### GitHub Pages → Cloudflare Pages

1. **Cloudflare Pages → Create a project → Connect to Git → 选择 Zip 仓库**
2. Build 设置同上
3. Pages 项目 → **Custom domains** → 添加原域名（Cloudflare 自动处理 DNS 和 SSL）
4. 按第 5 节配置 Access 认证
5. 整个过程 **< 15 分钟**，不影响原有 GitHub Actions 工作流

### Vercel → Cloudflare Pages

类似步骤，只是在 Cloudflare Pages 连接 Git 仓库后，Vercel 端的自动部署会自动失效（域名指向 Cloudflare 后）。

---

## 7. 进阶配置

### 7.1 自定义登录页外观

**Zero Trust → Access → Applications → 编辑应用 → Appearance**

- 标题、副标题、Logo
- 背景颜色、按钮样式
- 多语言支持（含中文）

### 7.2 备用管理员通道

防止误锁或邮箱失效：
1. 新增 Policy：`Action = Allow`，`Include = IPs → 填入家庭/办公公网 IP`
2. 将该策略置于最上方（优先级更高）
3. 即使邮箱验证失败，从指定 IP 访问仍可直通

### 7.3 审计与监控

- **Access Logs**：Zero Trust → Logs → Access — 查看所有登录尝试、成功/失败记录、来源 IP
- **Analytics**：Pages 项目 → Analytics — 查看认证后的页面访问量、热门笔记等

### 7.4 API/自动化访问

若需通过脚本（如 Airflow、SmartIDE）抓取笔记内容：
1. 在 Access Policy 中添加 **Service Token** 认证
2. 脚本请求时携带 Header：`CF-Access-Client-Id` + `CF-Access-Client-Secret`
3. 即可绕过交互式登录

### 7.5 Quartz 自定义美化

```bash
# 编辑自定义 CSS
nano quartz/styles/custom.scss
```

常用美化项：
- 修改字体、颜色主题
- 添加自定义页脚
- 启用/B 关闭特定 UI 组件

---

## 8. 注意事项

### 认证相关
1. ⚠️ **不要混用客户端鉴权**：Cloudflare Access 是网络层拦截，切勿在 Quartz 代码中添加 JS 登录检查作为唯一防护。两者定位完全不同。
2. ⚠️ **缓存问题**：开启认证后仍能直接访问？检查 Cloudflare Pages 的缓存设置，或在 **Rules → Cache Rules** 中对该域名禁用缓存。
3. 📊 **免费额度上限**：Free 计划限制 **50 个独立用户/月**。超出后需升级至 $3/用户/月的 Teams 计划，或改用 Vercel Pro ($20/月不限人数)。
4. 🔒 **敏感数据兜底**：即使有网关认证，仍建议对极度敏感的笔记使用 `obsidian-encrypt` 插件进行端到端加密，实现双重保险。

### 内容管理
- 使用 `ignorePatterns` 配置排除私密文件夹
- 定期 `git pull` 更新 Quartz 版本
- 修改 Obsidian 笔记后 push 到 GitHub 即自动触发构建

---

> 📌 最后更新：2026-07-07
