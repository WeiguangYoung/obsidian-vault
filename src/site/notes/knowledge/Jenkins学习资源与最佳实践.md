---
{"dg-publish":true,"permalink":"/knowledge/Jenkins学习资源与最佳实践/","tags":["Jenkins","CI/CD","DevOps","Pipeline","学习资源","最佳实践"],"dg-note-properties":{"date":"2026-07-06","tags":["Jenkins","CI/CD","DevOps","Pipeline","学习资源","最佳实践"]}}
---


# 🔧 Jenkins 学习资源与最佳实践

> Jenkins 是 CI/CD 领域的事实标准，无论是奇瑞的嵌入式 CICD、长鑫的 Agent 开发、智元的 DevOps 还是比亚迪的 Pipeline 岗位，都绕不开它。

---

## 1️⃣ 官方文档（最权威，首选）

| 资源 | 链接 | 说明 |
|:-----|:-----|:------|
| **Jenkins 用户手册** | `jenkins.io/doc/book/` | Pipeline、配置、安全，从头读到尾 |
| **Pipeline 语法参考** | `jenkins.io/doc/book/pipeline/syntax/` | Declarative + Scripted 完整语法 |
| **Pipeline 在线生成器** | `jenkins.io/pipeline-syntax/` | 可视化生成 Pipeline 代码片段 |
| **官方博客** | `jenkins.io/blog/` | 新版本发布、Feature 介绍 |
| **插件索引** | `plugins.jenkins.io/` | 查插件的文档、版本、兼容性 |
| **Javadoc** | `javadoc.jenkins.io/` | 写自定义插件/扩展时查 API |
| **Jenkins 官方 GitHub** | `github.com/jenkinsci` | 源码 + CI 示例 + Helm Charts |

---

## 2️⃣ 官方示例仓库（上手最快）

| 仓库 | 说明 |
|:-----|:------|
| **jenkinsci/pipeline-examples** ⭐ | GitHub 官方维护的 Pipeline 示例集，从 Hello World 到共享库、并行构建全有 |
| **jenkinsci/docker** | Jenkins 官方 Docker 镜像的 Dockerfile，参考如何容器化 |
| **jenkinsci/helm-charts** | Jenkins 在 Kubernetes 上的 Helm 部署方式 |
| **jenkinsci/configuration-as-code-plugin** | JCasC（Jenkins 配置即代码）完整示例 + 文档 |
| **jenkinsci/pipeline-library-template** | 共享库的项目模板，起步直接 fork |

---

## 3️⃣ 经典开源实践项目

| 项目 | 说明 | 技术亮点 |
|:-----|:------|:---------|
| **jenkins-infra/jenkins.io** | Jenkins 官网的 CI/CD 流水线源码 | 生产级 Jenkins 实际用法 |
| **jenkinsci/plugin-pom** | Jenkins 插件的父 POM 和 CI 配置 | Maven 多模块 + Jenkins Pipeline |
| **spring-projects/spring-petclinic** ⭐ | Spring 官方 Demo，有完整 Jenkinsfile | 适合第一次跑 Pipeline |
| **argoproj/argo-cd** | ArgoCD 的 CI 流水线 | Jenkins + Docker + K8s 完整链路 |
| **grafana/grafana** | Grafana CI 流水线（参考大型项目如何做） | 多阶段、矩阵构建、并行测试 |

---

## 4️⃣ 中文社区 & 博客

| 资源 | 说明 |
|:-----|:------|
| **Jenkins 中文社区** (`jenkins-zh.cn`) | 官方中文文档 + 本地化文章 |
| **❤️ 美团技术团队** | 搜索"Jenkins"或"CI/CD"→ 大型 CI/CD 体系实战案例 |
| **❤️ 腾讯云开发者社区** | DevOps + Jenkins + K8s 实践文章质量很高 |
| **❤️ 阿里云云栖社区** | Jenkins + 云原生 CI/CD 方案 |
| **❤️ 掘金** | 搜 `Jenkins Pipeline 最佳实践`，看高赞+近期文章 |
| **❤️ CSDN** | 搜 `Jenkins 实战`，筛选近一年文章 |

---

## 5️⃣ 英文优质博客 & 教程站

| 资源 | 说明 |
|:-----|:------|
| **DevOpsCube** (`devopscube.com`) | Jenkins 专栏，从入门到 K8s 集成，质量很高 |
| **Valaxy Technologies** | YouTube + 博客，Jenkins 从零到实战系列 |
| **Medium / tag:jenkins** | 全球一线工程师的实战分享 |
| **DZone / devops** | 大量 Jenkins 相关文章 |
| **Ivan Fadilla 博客** | Jenkins Shared Library 写得特别好 |

---

## 6️⃣ 必知核心概念 / 面试常问

```
分层掌握：

🟢 基础（必备）
    ├─ Pipeline: Declarative vs Scripted 区别
    ├─ Stage / Step / Agent 概念
    ├─ 环境变量与参数
    ├─ Post 条件 (always/success/failure)
    └─ 制品归档 (archiveArtifacts)

🟡 进阶（面试高频）
    ├─ Shared Library（共享库：vars/ + src/ 目录结构）
    ├─ Configuration as Code (JCasC: jenkins.yaml)
    ├─ Jenkins + Kubernetes 动态 Agent
    ├─ 多分支 Pipeline (Multibranch Pipeline)
    ├─ 并行构建 (parallel)
    ├─ 凭证管理 (Credential Binding / Vault)
    └─ Blue Ocean UI

🔴 高级（架构师向）
    ├─ Jenkins 高可用方案（冷备 / 热备 / 负载均衡）
    ├─ JVM 调优与 GC 配置
    ├─ 数据迁移 / Job 备份恢复
    ├─ 细粒度权限管理 (RBAC / Role Strategy)
    ├─ Pipeline 与 API 网关 / GitOps 集成
    └─ Jenkins + Terraform + Ansible 基础设施即代码
```

---

## 7️⃣ 学习路径建议

```
阶段一：跑起来 ──────────────── 1 周
    ├─ 用 Docker 跑一个 Jenkins
    ├─ 创建一个自由风格 Job
    └─ 创建第一个 Pipeline Job（Hello World）

阶段二：Pipeline 核心 ───────── 1~2 周
    ├─ 掌握 Declarative Pipeline 语法
    ├─ pipeline-examples 仓库 抄 5 个例子
    ├─ 集成 GitLab/GitHub Webhook 触发
    └─ 加入 SonarQube 代码扫描步骤

阶段三：工程化 ──────────────── 2~3 周
    ├─ 封装 Shared Library（通用工具函数）
    ├─ 搭建 Jenkins + GitLab + SonarQube + Nexus 流水线
    ├─ Configuration as Code 接管 Jenkins 配置
    └─ Docker 多阶段构建 + 制品推送

阶段四：生产化 ──────────────── 按需
    ├─ Jenkins on Kubernetes（Helm 部署 + 动态 Agent）
    ├─ GitOps (ArgoCD) 集成
    ├─ 安全加固与审计
    └─ 高可用部署
```

---

## 8️⃣ 推荐的实践项目

| 项目 | 技能覆盖 |
|:-----|:---------|
| **Docker Jenkins + Pipeline + SonarQube** | 容器化部署 + 代码质量门禁 |
| **Jenkins + GitLab Webhook + 自动部署** | 事件触发 + CD 流程 |
| **Shared Library 封装公司级 Pipeline** | Groovy 编程 + 抽象复用 |
| **JCasC 全配置 Jenkins Master** | 配置即代码 + 版本管理 |
| **Jenkins on K8s 动态 Agent** | K8s + Jenkins 集大门 |
| **Multibranch Pipeline + 版本号注入** | 分支策略 + Git 集成 |

---

## 9️⃣ 付费课程（需要时再看）

| 平台                    | 课程                                         | 适合            |
| :-------------------- | :----------------------------------------- | :------------ |
| **Udemy**             | Jenkins, From Zero To Hero                 | 零基础入门         |
| **Udemy**             | Jenkins Pipeline: Declarative and Scripted | Pipeline 专项强化 |
| **LinkedIn Learning** | DevOps with Jenkins                        | DevOps 体系化视角  |

---

## 🎯 Jenkins 常见面试题（按难度分级）

### 🟢 基础（初级工程师）

**Q1: Jenkins 是什么？和 GitLab CI / GitHub Actions 有什么区别？**

Jenkins 是一个开源的自动化服务器，用于 CI/CD。它是**自托管**的，需要自己维护服务器，优势是高度可定制（4000+ 插件），适合复杂的企业级流水线。GitLab CI / GitHub Actions 是 SaaS 平台的开箱即用方案，免运维，但定制能力有限。

**使用场景选择：**
| 场景 | 推荐 |
|:-----|:------|
| 简单项目、团队小 | GitLab CI / GitHub Actions |
| 复杂流水线、企业合规 | Jenkins |
| 需要 Jenkins 独家插件 | Jenkins |
| 不想管服务器 | SaaS |

---

**Q2: Jenkins Master 和 Agent 是什么？**

- **Master（控制器）**：负责调度任务、管理配置、提供 UI 和 API
- **Agent（执行器/节点）**：实际执行构建、测试、部署任务的 Worker

Master 只负责任务调度和结果收集，Agent 负责具体干活。可以有多台 Agent 实现并行构建。

---

**Q3: Declarative Pipeline vs Scripted Pipeline 的区别？**

| 对比 | Declarative | Scripted |
|:-----|:-----------|:--------|
| **语法风格** | 结构化、声明式（`pipeline { }`） | 自由编程式（`node { }`） |
| **学习曲线** | 低，模板化 | 高，类 Groovy 编程 |
| **错误检查** | 编译时检查 | 运行时检查 |
| **内置功能** | `post`、`options`、`when`、`environment` 内建 | 需手动实现 |
| **推荐** | ✅ 90% 场景用这个 | 复杂逻辑时用 |

**核心区别一句话：Declarative 限制了你的写法但不容易出错，Scripted 给你全部自由但自己负责。**

---

**Q4: Pipeline 的基本结构写一下**

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                sh 'make'
            }
        }
        stage('Test') {
            steps {
                sh 'make test'
            }
        }
        stage('Deploy') {
            steps {
                sh 'deploy.sh'
            }
        }
    }

    post {
        success { echo '构建成功!' }
        failure { echo '构建失败!' }
    }
}
```

**核心元素：** `pipeline` → `agent` → `stages` → `stage` → `steps` → `post`

---

**Q5: agent 有哪些类型？**

```groovy
agent any                     // 任意可用 Agent
agent none                    // 全局不指定，每个 stage 各自指定
agent { label 'linux' }       // 按标签选择
agent { docker 'node:18' }    // 在 Docker 容器中执行
agent { 
    kubernetes {
        yaml podTemplate // 在 K8s Pod 中动态创建
    }
}
```

---

### 🟡 进阶（2~3 年经验）

**Q6: Shared Library 是什么？怎么用？**

Shared Library 是把通用的 Pipeline 逻辑封装成可复用的代码库，放在 `vars/` 和 `src/` 目录下。

**目录结构：**
```
jenkins-shared-library/
├── vars/
│   ├── dockerBuild.groovy       # 全局函数（可直接在 Pipeline 中调用）
│   └── notifyWechat.groovy
├── src/
│   └── com/company/
│       └── PipelineUtils.groovy # 普通 Groovy 类
└── resources/
    └── templates/
        └── email.html
```

**使用方式：**
```groovy
// Jenkinsfile
@Library('my-shared-library')_

dockerBuild('my-app', '1.0.0')
notifyWechat(status: 'success')
```

**面试常问：** `@Library` 后面的 `_` 是什么意思？—— 下划线是 Groovy 的占位符，用于导入 Library 中的符号到全局命名空间。

---

**Q7: Jenkins 怎么实现配置即代码 (JCasC)？**

JCasC（Jenkins Configuration as Code）通过 YAML 文件声明式地定义 Jenkins 配置：

```yaml
# jenkins.yaml
jenkins:
  systemMessage: "Jenkins 由 JCasC 管理"
  numExecutors: 2
  scm:
    git:
      globalConfigName: "jenkins"
      globalConfigEmail: "jenkins@company.com"

tool:
  git:
    installations:
      - name: "Default"
        home: "/usr/bin/git"
  maven:
    installations:
      - name: "M3"
        properties:
          - installSource:
              installers:
                - maven:
                    id: "3.9.6"
```

**优势：** 所有配置可版本化管理 → 新 Jenkins 实例一键恢复 → 消除手动配置的漂移。

---

**Q8: Multibranch Pipeline 是什么？**

Multibranch Pipeline 会自动发现代码仓库中的每个分支（以及 PR/MR），为每个分支创建独立的 Pipeline。

```groovy
// 每个分支的 Jenkinsfile 通用，但分支名可用环境变量获取
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo "分支: ${env.BRANCH_NAME}"
                echo "提交: ${env.GIT_COMMIT}"
            }
        }
    }
}
```

**分支策略实践：**
| 分支类型 | 流水线动作 |
|:---------|:-----------|
| `feature/*` | 仅编译+单元测试 |
| `develop` | 编译+测试+集成测试 |
| `release/*` | 编译+测试+构建制品+发布 |
| `main/master` | 全流程+生产部署 |

**触发方式：** Webhook（GitHub/GitLab push 事件 → Jenkins）、定时扫描仓库

---

**Q9: Jenkins + Kubernetes 动态 Agent 的原理？**

不再需要固定数量的 Agent 机器，每次构建按需创建 Pod，构建完自动销毁：

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest
  - name: golang
    image: golang:1.21
    command: ["cat"]
    tty: true
  - name: docker
    image: docker:24
    command: ["cat"]
    tty: true
    volumeMounts:
    - name: docker
      mountPath: /var/run/docker.sock
  volumes:
  - name: docker
    hostPath:
      path: /var/run/docker.sock
"""
        }
    }

    stages {
        stage('Test') {
            steps {
                container('golang') {
                    sh 'go test ./...'
                }
            }
        }
        stage('Build Image') {
            steps {
                container('docker') {
                    sh 'docker build -t myapp .'
                }
            }
        }
    }
}
```

**优势：** 弹性伸缩、资源利用率高、环境隔离、构建完即销毁。

---

**Q10: Pipeline 中怎么处理凭证（Credentials）？**

```groovy
// 方法1：内置凭据绑定
withCredentials([
    usernamePassword(
        credentialsId: 'gitlab-cred',
        usernameVariable: 'GIT_USER',
        passwordVariable: 'GIT_PASS'
    ),
    string(
        credentialsId: 'slack-token',
        variable: 'SLACK_TOKEN'
    )
]) {
    sh 'git push https://$GIT_USER:$GIT_PASS@gitlab.com/...'
}

// 方法2：SSH 密钥
sshagent(['gitlab-ssh-key']) {
    sh 'git push origin main'
}
```

**注意：** 永远不要把密码写在 Jenkinsfile 里，用 Credential Binding 或 Vault 集成。

---

**Q11: 流水线中如何并行执行任务？**

```groovy
pipeline {
    agent any
    stages {
        stage('Parallel Tests') {
            parallel {
                stage('Unit Tests') {
                    steps { sh 'make unit-test' }
                }
                stage('Integration Tests') {
                    steps { sh 'make integration-test' }
                }
                stage('Lint') {
                    steps { sh 'make lint' }
                }
            }
        }
    }
    post {
        always {
            // 即使某个并行分支失败，其他分支继续执行
            junit '**/reports/*.xml'
        }
    }
}
```

**关键参数：** `failFast true` — 任何一个分支失败就立即终止其他分支。

---

**Q12: post 条件有哪几种？**

| 条件 | 触发时机 |
|:-----|:---------|
| `always` | 无论结果如何都执行 |
| `success` | 构建成功 |
| `failure` | 构建失败 |
| `unstable` | 构建不稳定（测试失败但编译通过） |
| `changed` | 本次结果与前一次不同 |
| `aborted` | 用户手动取消 |
| `regression` | 从成功变为失败 |
| `fixed` | 从失败变为成功 |

---

**Q13: 什么是 Pipeline 的 SCM Polling vs Webhook 触发？**

- **Polling（轮询）：** Jenkins 定期（如每分钟）检查 Git 仓库是否有新提交，低效、有延迟
- **Webhook（推送）：** Git 仓库有 push/PR 时主动通知 Jenkins，实时、高效

```groovy
// Polling 方式（不推荐）
pipeline {
    triggers {
        pollSCM('*/5 * * * *') // 每 5 分钟检查一次
    }
}

// Webhook 方式（推荐）
pipeline {
    triggers {
        // 不需要 pollSCM，在 GitLab/GitHub 中配置 Webhook URL
    }
}
```

---

### 🔴 高级/架构师（3 年+ / 负责人）

**Q14: Jenkins 高可用怎么实现？**

| 方案 | 说明 | 适用场景 |
|:-----|:------|:---------|
| **冷备** | 定期备份 JENKINS_HOME，故障时手动恢复 | 非关键系统 |
| **热备 + 共享存储** | 主备共享 JENKINS_HOME（NFS），Keepalived 做浮动 IP | 准生产 |
| **Active/Active + HA 插件** | 多 Master 共享数据库 + 共享存储 | 大型生产 |
| **Jenkins on K8s** | K8s StatefulSet + PV 持久化，Pod 挂了自动重建 | 云原生最佳方案 |

**核心难点：** Jenkins 的 JENKINS_HOME 里包含大量运行时状态（构建历史、制品、日志），热备切换时需要保证数据一致性。

---

**Q15: JENKINS_HOME 里都有些什么？**

```
JENKINS_HOME/
├── jobs/               # 所有 Job 的配置和构建记录
├── plugins/            # 已安装的插件及其配置
├── config.xml          # Jenkins 全局配置（JCasC 覆盖）
├── secrets/            # 凭据密钥（加密存储）
├── users/              # 用户配置
├── nodes/              # Agent 节点配置
├── updates/            # 插件更新信息
├── war/                # Jenkins Web 应用
├── fingerprint/        # 文件指纹（产物追踪用）
└── logs/               # 日志
```

**备份重点：** `jobs/`、`plugins/`、`config.xml`、`secrets/`。`builds/` 下构建记录体积很大，可根据需要排除。

---

**Q16: Jenkins Pipeline 中的 `when` 指令有哪些常用条件？**

```groovy
stage('Deploy') {
    when {
        branch 'main'           // 只在 main 分支执行
        expression {            // 自定义 Groovy 表达式
            return env.BRANCH_NAME ==~ /release\/.*/
        }
        changeset '**/*.java'   // 只当 Java 文件有变更时
        environment name: 'BUILD_TYPE', value: 'release'
        beforeAgent true        // 在分配 Agent 前就判断（节省资源）
        allOf {                 // 所有条件同时满足
            branch 'main'
            triggeredBy 'SCMTrigger'
        }
        anyOf {                 // 满足任一条件
            branch 'main'
            branch 'develop'
        }
        not {                   // 取反
            tag '*-alpha'
        }
    }
    steps {
        sh './deploy.sh'
    }
}
```

---

**Q17: Jenkins 安全怎么加固？**

| 层面 | 措施 |
|:-----|:------|
| **认证** | LDAP/SSO 集成，禁用默认用户 |
| **授权** | RBAC（Role-Based Strategy 插件） |
| **凭证** | 使用 Credential Provider（Vault）替代明文 |
| **Agent** | Agent 和 Master 之间走 SSH/WSS 加密通道 |
| **脚本安全** | Groovy Sandbox 限制 Pipeline 脚本 |
| **CSRF 保护** | 启用 CSRF 令牌（默认） |
| **Agent 隔离** | K8s 动态 Agent → 每个构建独立 Pod |
| **审计** | Audit Trail 插件记录所有操作 |
| **更新** | 定期更新 Jenkins + 插件 |

---

**Q18: Jenkins 性能调优怎么做？**

```bash
# JVM 调优（默认 256MB heap 太小）
export JAVA_OPTS="-Xms2g -Xmx4g \
  -XX:MaxMetaspaceSize=512m \
  -XX:+UseG1GC \
  -XX:+ParallelRefProcEnabled \
  -XX:+DisableExplicitGC"
```

**其他优化：**
| 问题 | 解决 |
|:-----|:------|
| **构建队列堆积** | 增加 Agent 数量 / 改用 K8s 动态 Agent |
| **Pipeline 执行慢** | 减少层数、并行化 stage、避免重复 checkout |
| **日志过大** | 设置 log rotator：`options { buildDiscarder(logRotator(numToKeepStr: '20')) }` |
| **插件太多** | 只装必要的插件，禁用不用的 |
| **构建历史无限增长** | 配置 Job 保留策略（保留近 30 天或近 50 次构建） |
| **GC 频繁/Full GC** | 加大 -Xmx，切 G1GC |

---

**Q19: 怎么实现 Pipeline 的失败重试和错误处理？**

```groovy
// 方法1：retry — 重试指定次数
stage('Flaky Test') {
    retry(3) {
        sh './run-flaky-test.sh'
    }
}

// 方法2：timeout — 超时控制
stage('Build') {
    timeout(time: 10, unit: 'MINUTES') {
        sh 'make'
    }
}

// 方法3：try-catch 自定义错误处理
stage('Deploy') {
    steps {
        script {
            try {
                sh './deploy.sh'
            } catch (Exception e) {
                echo "部署失败: ${e.message}"
                // 通知运维
                currentBuild.result = 'UNSTABLE'
            }
        }
    }
}

// 方法4：post 条件处理（推荐）
post {
    failure {
        emailext(
            subject: "构建失败: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
            body: "请检查构建日志",
            to: "team@company.com"
        )
    }
}
```

---

**Q20: env、params、currentBuild 的区别？**

| 变量 | 来源 | 示例 |
|:-----|:-----|:-----|
| `env` | Jenkins 自动设置的环境变量 | `env.BRANCH_NAME`, `env.BUILD_NUMBER` |
| `params` | Pipeline 定义的参数 | `params.TAG`, `params.DEPLOY_ENV` |
| `currentBuild` | 当前构建对象 | `currentBuild.result`, `currentBuild.duration` |

```groovy
pipeline {
    parameters {
        string(name: 'TAG', defaultValue: 'latest', description: '镜像标签')
        choice(name: 'ENV', choices: ['dev', 'staging', 'prod'], description: '部署环境')
    }
    stages {
        stage('Deploy') {
            steps {
                echo "构建 #${env.BUILD_NUMBER}"
                echo "部署 ${params.TAG} 到 ${params.ENV}"
                currentBuild.displayName = "${params.ENV}-${env.BUILD_NUMBER}"
            }
        }
    }
}
```

---

### 💡 面试加分题

**Git flow vs Trunk Based Development 在 Jenkins 中的差异？**

| 分支策略 | Pipeline 配置 | 适用场景 |
|:---------|:-------------|:---------|
| **Git Flow** | 多层 Pipeline：feature→develop→release→main，每层不同的 CI 力度 | 大团队、固定发布周期 |
| **Trunk Based** | 单一 Pipeline，所有人在 main 上高频合并 | 小团队、持续部署、敏捷开发 |

**Jenkins 和 ArgoCD 的关系？**
- Jenkins 负责 **CI**：构建、测试、打包镜像
- ArgoCD 负责 **CD**：把镜像部署到 K8s，维持集群状态与 Git 仓库一致
- Jenkins Pipeline 最后一步触发 ArgoCD 同步，或 ArgoCD 自动检测新镜像

**Docker 容器中运行 Jenkins 的注意事项？**
1. `/var/run/docker.sock` 挂载实现 DIND（Docker in Docker）
2. JENKINS_HOME 要挂载持久卷
3. Agent 用 Docker 容器启动（不再是 SSH）
4. 重启策略：`--restart=unless-stopped`
5. 日志限制：`--log-opt max-size=10m --log-opt max-file=3`

---

### 📊 面试题与岗位匹配

| 岗位 | 重点关注 |
|:-----|:---------|
| **奇瑞 CICD** | Pipeline 基础、Agent 管理、SonarQube 集成、制品归档、构建环境维护 |
| **智元 DevOps** | K8s 动态 Agent、多集群部署、环境变量/凭证管理，高可用 |
| **比亚迪 Pipeline** | Shared Library、JCasC、并行执行、参数化构建、触发策略 |
| **长鑫 Agent** | Pipeline 自动化集成、代码质量门禁、构建效率优化 |

---

## 📂 关联文件

- 奇瑞 CICD 岗位（嵌入式 CI/CD）：`../job/奇瑞汽车-CICD工程师/奇瑞汽车-CICD工程师.md`
- 智元机器人 DevOps（CI/CD + K8s）：`../job/智元机器人-DevOps平台工程师/智元机器人-DevOps平台工程师.md`
- 比亚迪 Pipeline（工作流引擎+CI/CD）：`../job/比亚迪-Pipeline开发工程师/比亚迪-Pipeline开发工程师.md`
- Docker 底层原理（容器化 Jenkins 的基础）：`Docker底层原理入门.md`

---

> 🦐 虾管家 · 2026-07-06
