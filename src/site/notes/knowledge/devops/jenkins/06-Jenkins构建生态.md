---
{"dg-publish":true,"permalink":"/knowledge/devops/jenkins/06-Jenkins构建生态/","tags":["Jenkins","CI/CD","DevOps","Credentials","Artifacts","Plugin"],"dg-note-properties":{"date":"2026-09-16","tags":["Jenkins","CI/CD","DevOps","Credentials","Artifacts","Plugin"]}}
---


# 🧩 Jenkins 构建生态

> Jenkins 功能模块全景 · 第六章。构建跑起来之后，配套的密钥、产物、插件、工具、工作区。

---

## 一、Credentials（凭证管理）⭐

**统一存放敏感信息**，流水线里用 ID 引用，明文不落 Jenkinsfile。

| 类型 | 用途 |
|:-----|:-----|
| Username with password | 账号密码（Git、仓库、Registry 登录） |
| SSH Username with private key | SSH Key（Git over SSH、远程部署） |
| Secret text | Token（API token、Webhook secret） |
| Secret file | 证书、kubeconfig 等文件 |
| Certificate | 客户端证书（PKCS#12） |

**用法**：

```groovy
withCredentials([usernamePassword(
        credentialsId: 'git-cred',
        usernameVariable: 'GIT_USER',
        passwordVariable: 'GIT_PASS')]) {
    sh 'git push https://$GIT_USER:$GIT_PASS@repo.git'
}

sshagent(['deploy-key']) {
    sh 'ssh user@host "deploy.sh"'
}
```

- **作用域**：全局 / Folder 级 / 任务级
- **铁律**：永远不要把密码写进 Jenkinsfile，用 Credential Binding
- **进阶**：接 HashiCorp Vault 做集中密钥管理

## 二、Artifacts（制品归档）

构建产物（.jar / .apk / .exe / 镜像 tar / 报告）的归档与追溯。

```groovy
archiveArtifacts artifacts: 'build/*.jar', fingerprint: true
```

- **fingerprint**：记录制品的"指纹"，可追溯"这个 jar 是哪个构建、哪次提交产出的"
- 归档物存在 `$JENKINS_HOME/jobs/<job>/builds/<n>/archive/`
- **注意**：小制品放 Jenkins 没问题；大制品/镜像应推**制品仓库**（Nexus / Harbor / ACR），Jenkins 只存引用

## 三、Plugins（插件体系）⭐

Jenkins 的能力边界基本由插件决定，官方 + 社区共 **1800+**。

| 领域 | 代表插件 |
|:-----|:-----|
| SCM | Git、GitHub、GitLab、Gitea、Bitbucket |
| 构建工具 | Maven、Gradle、NodeJS、Docker、Kubernetes |
| 测试/质量 | JUnit、JaCoCo、Cobertura、SonarQube、cppcheck、Warnings NG |
| 通知 | Email、Slack、企业微信、钉钉 |
| 触发 | Generic Webhook Trigger、Multibranch |
| 运维 | ThinBackup、Configuration as Code、Role-based Authorization |

- **安装**：UI 在线装 / 手动上传 .hpi / CLI
- **⚠️ 坑**：插件有**依赖链**，装 A 可能要求先装 B；版本不兼容会导致 Jenkins 起不来——升级前先备份
- **最小化原则**：只装需要的，插件越多越容易冲突、越慢

## 四、工具自动安装（Tool Auto-Install）

让 Jenkins 按需把 JDK / Maven / Gradle / Node 等**自动装到 Agent**，无需手工维护。

- 配置处：`Manage Jenkins → Tools`
- 流水线里用 `tools { maven 'M3' }` 引用
- 好处：环境一致、Agent 无需预装
- **替代路线**：工具链打进 Docker 镜像，用 `agent { docker }` 拉取（**更推荐、更可复现**）

## 五、Workspace（工作区）

每个任务/流水线在 Agent 上的工作目录，源码 checkout、构建都在这里。

- 路径：`$JENKINS_HOME/workspace/<job>` 或 Agent 上的自定义目录
- **复用**：`reuseNode true` 让同 stage 内复用同一节点工作区
- **共享挂载**：多个阶段需共享文件时，注意工作区是否在同一节点
- **⚠️ 常见坑**：
  - **动态 Agent 的工作区是临时的**（Pod 销毁即丢），跨构建缓存会失效 → 冷构建变慢
  - 工作区不清理会撑爆磁盘 → 配 `cleanWs()` 或定期清理
  - **dockerfile agent 下 workspace 映射错位**会导致构建僵尸

## 六、一张图串起来

```
        代码库 (Git)
           │ checkout
           ▼
   ┌─────────────────┐
   │   Workspace     │  ← 工作区，干活的地方
   │  (Agent 上)      │
   └────────┬────────┘
            │ 用工具链构建
            ▼
   ┌─────────────────┐     ┌──────────────┐
   │  Artifacts 归档  │────▶│ 制品仓库       │
   └─────────────────┘     │ Nexus/Harbor │
            │               └──────────────┘
            │ 凭 Credentials 推送/部署
            ▼
        目标环境
```

- **Credentials** 管"钥匙"
- **Workspace** 是"车间"
- **工具自动安装 / Docker 镜像** 提供"机床"
- **Artifacts** 是"成品"及其追溯
- **Plugins** 决定"能接什么外延"

> 💡 **实战三条**：① 密钥一律走 Credentials/Vault，绝不硬编码；② 大制品进制品库，Jenkins 只留指纹；③ 构建环境优先用 Docker 镜像固化，别依赖 Agent 预装。

---

> 🦐 虾管家 · 2026-09-16
