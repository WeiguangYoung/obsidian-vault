---
{"dg-publish":true,"permalink":"/knowledge/devops/jenkins/05-Jenkins触发机制/","tags":["Jenkins","CI/CD","DevOps","触发器","Webhook"],"dg-note-properties":{"date":"2026-09-16","tags":["Jenkins","CI/CD","DevOps","触发器","Webhook"]}}
---


# 🧩 Jenkins 触发机制

> Jenkins 功能模块全景 · 第五章。流水线怎么被"叫起来"跑。

---

## 一、SCM 轮询（Poll SCM）

Controller 按 cron 定时去查代码库有没有新提交，有就触发。

```groovy
triggers {
    pollSCM('H/5 * * * *')   // 每 5 分钟查一次
}
```

- **优点**：不需要代码库配合，配置简单
- **缺点**：有延迟（最多一个轮询周期）、空轮询浪费资源
- **建议**：能用 Webhook 就别用轮询

## 二、Webhook（最推荐）

代码库在**有 push/MR 事件时主动通知** Jenkins，实时、零延迟。

**流程**：
```
开发者 push / 提 MR
      │
      ▼
代码库 (Gitea/GitHub/GitLab) ──HTTP POST──▶ Jenkins /generic-webhook-trigger 或 /gitea-webhook
                                              │
                                              ▼
                                          触发构建
```

**要点**：
- Jenkins 侧需开 **"触发远程构建"** 或装 **Gitea/GitHub/GitLab 插件**、**Generic Webhook Trigger**
- 供 Webhook 调用的地址必须**公网可达**（或经反向代理/内网穿透）
- 多分支流水线（Multibranch）用 **Branch Source** 插件，Webhook 打的是 Controller 的扫描接口
- **⚠️ 常见坑**：Webhook 通了但构建没触发 → 检查 ① token 对不对 ② 分支源扫描有没有发现该分支 ③ 插件版本

## 三、定时构建（Build periodically）

按 cron 语法定时跑，不管有没有代码变化。

```groovy
triggers {
    cron('0 2 * * *')        // 每天凌晨 2 点
    cron('H 4 * * 1-5')      // 工作日 4 点左右（H 表示错峰）
}
```

- 用途：**Nightly Build**（每晚全量构建 + 全量扫描）、定时报告
- **`H` 哈希**：让时间点分散，避免整点所有任务齐跑把 Controller 打爆

## 四、上游任务触发（Build after other projects）

A 构建完自动触发 B，形成流水线链。

```groovy
triggers {
    upstream(upstreamProjects: 'job-A', threshold: hudson.model.Result.SUCCESS)
}
```

## 五、远程触发（触发远程构建）

外部通过带 token 的 URL 直接触发（脚本、其他系统调用）：

```bash
curl -X POST http://jenkins/job/我的任务/build?token=MY_TOKEN
# 带参数
curl -X POST "http://jenkins/job/我的任务/buildWithParameters?PARAM=value&token=MY_TOKEN"
```

- 适合：被外部系统（发布平台、调度器）拉起

## 六、手动触发 & 参数化

- UI 上点「立即构建」
- **参数化构建**：预先定义参数（字符串/选择/布尔/凭证），跑前填写

```groovy
parameters {
    string(name: 'VERSION', defaultValue: '1.0.0', description: '版本号')
    choice(name: 'ENV', choices: ['dev', 'test', 'prod'], description: '环境')
    booleanParam(name: 'PUSH_IMAGE', defaultValue: false, description: '是否推镜像')
}
```

- 发布场景常用：手动选分支 / 填版本号 / 勾是否推镜像

## 七、触发方式对比

| 方式 | 实时性 | 需要代码库配合 | 需要公网 | 典型场景 |
|:-----|:---:|:---:|:---:|:-----|
| SCM 轮询 | 低（有延迟） | ❌ | ❌ | 兜底、无 Webhook 条件 |
| **Webhook** | 高（实时） | ✅ | ✅ | 提交即触发、MR 门禁 |
| 定时 cron | — | ❌ | ❌ | 夜间全量、定时任务 |
| 上游任务 | 高 | ❌ | ❌ | 任务链 |
| 远程 URL | 高 | ❌ | ✅ | 外部系统拉起 |
| 手动/参数化 | — | ❌ | ❌ | 发布、按需 |

## 八、选择建议

```
提交后要马上跑？          → Webhook（首选）
代码库不支持 Webhook？    → SCM 轮询兜底
每晚要跑全量？            → 定时构建
任务之间有依赖？          → 上游任务触发
外部系统要拉起构建？      → 远程触发 URL
发布要人工选参数？        → 参数化 + 手动
```

> 💡 **实践共识**：**Webhook 为主、cron 为辅**。定时任务用 `H` 错峰，别都卡在整点；参数化构建把「推镜像/选环境」这类危险动作显式化，避免误触。

---

> 🦐 虾管家 · 2026-09-16
