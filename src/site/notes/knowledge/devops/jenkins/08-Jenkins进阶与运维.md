---
{"dg-publish":true,"permalink":"/knowledge/devops/jenkins/08-Jenkins进阶与运维/","tags":["Jenkins","DevOps","JCasC","备份","高可用","RESTAPI"],"dg-note-properties":{"date":"2026-09-16","tags":["Jenkins","DevOps","JCasC","备份","高可用","RESTAPI"]}}
---


# 🧩 Jenkins 进阶与运维

> Jenkins 功能模块全景 · 第八章。让 Jenkins 可维护、可迁移、可扩展的运维面能力。

---

## 一、Configuration as Code（JCasC）⭐

**用 YAML 定义 Jenkins 配置**，把系统配置也代码化、进版本库。

- 插件：**Configuration as Code**（`jenkins.yaml`）
- 覆盖：系统设置、凭证（引用外部源）、插件列表、节点、任务、安全域…

```yaml
jenkins:
  systemMessage: "Managed by JCasC"
  numExecutors: 0
  securityRealm:
    local:
      allowsSignup: false
  nodes:
    - permanent:
        name: "agent-01"
        remoteFS: "/home/jenkins/agent"
        launcher:
          jnlp: {}
credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              id: "git-cred"
              username: "ci"
              password: "${GIT_PASSWORD}"
```

- **价值**：Jenkins 挂了，一条 JCasC 重装回来；配置变更可评审、可回滚
- **配合**：Docker 镜像 + JCasC = 可复现的 Jenkins

## 二、REST API

Jenkins 全功能都有 HTTP API，可被外部系统集成：

```bash
# 触发构建
curl -X POST http://jenkins/job/myjob/build?token=XXX
# 查询构建状态
curl http://jenkins/job/myjob/lastBuild/api/json
# 获取控制台日志
curl http://jenkins/job/myjob/lastBuild/consoleText
# 创建/更新任务（用 config.xml）
curl -X POST http://jenkins/createItem?name=myjob \
     -H "Content-Type: application/xml" --data-binary @config.xml
```

- 认证：API Token（用户 → 配置 → API Token）或 Basic Auth
- 用途：看板取数、外部调度、自动化批量管理

## 三、CLI（命令行）

`jenkins-cli.jar`，通过 SSH 或 HTTP 通道操作 Jenkins：

```bash
java -jar jenkins-cli.jar -s http://jenkins/ -auth user:token list-jobs
java -jar jenkins-cli.jar -s http://jenkins/ build myjob -f -v
java -jar jenkins-cli.jar -s http://jenkins/ groovy = < script.groovy
```

- **`groovy =`**：直接对 Jenkins 内部对象执行 Groovy，能力极强（也是"管理员后门"，慎用）

## 四、安全域与授权

| 维度 | 选项 |
|:-----|:-----|
| **安全域**（你是谁） | Jenkins 本地库、LDAP、Active Directory、OIDC / OAuth、SAML |
| **授权**（你能干什么） | Matrix-based（细粒度矩阵）、Role-based（RBAC，角色策略）、项目矩阵、登录用户可做任何事 |

- **生产建议**：接 **LDAP/OIDC**（统一身份），授权用 **RBAC 或 Matrix**（最小权限）
- **安全红线**：
  - Controller 的 `/script` Groovy 控制台 = **root 权限入口**，必须锁死
  - 匿名用户绝不能有写权限
  - API Token 定期轮换

## 五、备份与迁移 ⭐

Jenkins 的全部状态都在 **`$JENKINS_HOME`**：

```
$JENKINS_HOME/
├── config.xml        系统配置
├── jobs/             所有任务定义 + 构建历史
├── plugins/          已装插件
├── secrets/          密钥、凭证加密文件
├── users/            用户
└── nodes/            节点配置
```

**备份要点**：重点备 `jobs/`、`plugins/`、`config.xml`、`secrets/`。`builds/` 历史体积大，可按需排除。

| 方式 | 说明 |
|:-----|:-----|
| 直接打包 JENKINS_HOME | 简单粗暴，停服或热备 |
| **ThinBackup 插件** | 增量备份，可定时、可排除大目录 |
| 迁移 | 新机器装同版本 Jenkins → 覆盖 JENKINS_HOME → 重启 |

- **⚠️ 版本必须一致**：跨大版本直接搬 JENKINS_HOME 容易起不来
- **⚠️ `secrets/` 不备 = 凭证全废**：密钥文件丢了，所有凭证解不开

## 六、高可用与水平扩展

Jenkins Controller **本身是单点**（原生不支持多活），扩展主要靠 Agent：

| 方向 | 做法 |
|:-----|:-----|
| **Agent 池** | 多台 Agent，任务排队自动分派 |
| **K8s 动态伸缩** | kubernetes-plugin 按需拉 Pod，闲时归零 |
| **Cloud 插件** | K8s / EC2 / Docker / 阿里云等动态供给节点 |
| **Controller 高可用** | 冷备 / 主备切换（社区方案），或迁到 K8s 用 PV 持久化 + 快速重建 |
| **Controller 负载** | `numExecutors: 0`（不跑构建）、分离高负载任务到 Agent |

- **K8s 上的 Controller**：Deployment + PVC（JENKINS_HOME）+ 定期 rsync 冷备
- **伸缩本质**：**Controller 定规模，Agent 定弹性**

## 七、可扩展性（自己造轮子）

| 方式 | 说明 |
|:-----|:-----|
| **插件开发** | Java + Jenkins API，可加 SCM/Trigger/Builder/Publisher 等扩展点 |
| **Shared Library** | Groovy 写自定义 Pipeline Step（`vars/` `src/`），不用发插件 |
| **Groovy 脚本** | `/script` 控制台或 `groovy =` CLI，直接操作内部对象 |
| **扩展点** | SCM、Trigger、Cloud、BuildWrapper、Publisher、Listener… |

## 八、运维 Checklist

```
□ Controller numExecutors = 0（不跑构建）
□ JENKINS_HOME 有定期备份（含 secrets/）
□ 接统一身份（LDAP/OIDC）+ RBAC 最小权限
□ /script 控制台锁死
□ API Token 定期轮换
□ Agent 池够用，动态 Agent 按需伸缩
□ 插件最小化，升级前备份
□ 配置尽量 JCasC 化，可重建
□ 磁盘监控（workspace 不清理会爆盘）
```

> 💡 **一句话**：Jenkins 运维的核心是——**配置代码化（JCasC）、状态可备份（JENKINS_HOME）、执行可弹性（Agent）、权限最小化（RBAC）**。

---

> 🦐 虾管家 · 2026-09-16
