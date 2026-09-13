---
{"dg-publish":true,"permalink":"/knowledge/devops/jenkins/01-Jenkins知识问答/","tags":["Jenkins","CI/CD","DevOps","Pipeline","知识问答","原理"],"dg-note-properties":{"date":"2026-07-06","tags":["Jenkins","CI/CD","DevOps","Pipeline","知识问答","原理"]}}
---


# 💡 Jenkins 知识问答

> Jenkins 原理性内容问答 + 实战的理论基础。适合复习原理、应对面试、理解 CI/CD 底层逻辑。
> 涉及：Pipeline 语法与执行模型、Master/Agent 架构、K8s 动态 Agent、配置即代码、凭证/安全/高可用等。

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

> 🦐 虾管家 · 2026-07-06

