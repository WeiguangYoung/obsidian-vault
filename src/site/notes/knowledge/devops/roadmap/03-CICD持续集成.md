---
{"dg-publish":true,"permalink":"/knowledge/devops/roadmap/03-CICD持续集成/","tags":["DevOps","CI/CD","Pipeline","Jenkins","GitLab","发布策略","多环境"],"dg-note-properties":{"date":"2026-07-19","tags":["DevOps","CI/CD","Pipeline","Jenkins","GitLab","发布策略","多环境"]}}
---


# 三、CI/CD 持续集成交付

> 从"代码提交"到"部署上线"这条流水线，怎么自动化、怎么控风险。

## 3.1 流水线平台

CI/CD 平台是这条流水线的"发动机"。选型时主要看两点：**代码托管在哪**（决定集成成本），**流程有多复杂**（决定灵活性需求）。

| 平台 | 特点 | 适用场景 |
|:----|:------|:------|
| **Jenkins** | 最灵活，插件生态丰富，Pipeline as Code (Groovy) | 自建平台、复杂流程、嵌入式/汽车行业 |
| **GitLab CI** | 与 GitLab 深度集成，.gitlab-ci.yml 声明式配置 | 源码在 GitLab、中小团队 |
| **GitHub Actions** | GitHub 原生，Marketplace 生态好 | 开源项目、GitHub 为主的技术栈 |

Jenkins 的优势是"什么都能干"——尤其适合需要自定义构建环境（比如交叉编译、License 绑定的 Windows IDE）的场景，这也是嵌入式/汽车行业偏爱它的原因。

### CI / CD / CD：三个词的区别

这三个缩写常被混淆，其实是一条链上的三个位置：

- **CI（持续集成）**：开发者频繁把代码合入主干，每次合入都**自动构建 + 跑测试**，目的是"尽早发现集成问题"——不要等到发布前才发现三周前的改动冲突了。
- **CD（持续交付，Continuous Delivery）**：在 CI 基础上，保证**任何时刻都能一键发布**——但最后一步上线要**人工审批**。
- **CD（持续部署，Continuous Deployment）**：把最后一步也自动化，**过了门禁就自动发到生产**。

一句话记忆：**Delivery 与 Deployment 的区别，就在"最后一步是否人工"**。

### Pipeline as Code

把流水线定义写进代码仓库（Jenkinsfile / .gitlab-ci.yml），而不是在界面上点。好处很直接：**可评审**（走 Code Review）、**可追溯**（跟着代码版本走）、**可复用**（抽成模板）。

```groovy
// Jenkins 声明式 Pipeline
pipeline {
    agent any
    stages {
        stage('Checkout') { steps { git url: '...' } }
        stage('Build')    { steps { sh 'make' } }
        stage('Test')     { steps { sh './run_tests.sh' } }
        stage('Deploy')   { steps { sh 'kubectl apply -f deploy/' } }
    }
}
```

```yaml
# GitLab CI
stages: [build, test, deploy]

build:
  stage: build
  script: make
  artifacts:
    paths: [build/]        # 产物传给后续 stage

test:
  stage: test
  script: ./run_tests.sh

deploy:
  stage: deploy
  script: kubectl apply -f deploy/
  only: [main]             # 只在 main 分支执行
```

## 3.2 流水线设计

一条标准的 CI/CD 流水线，本质是**一串"检查点"**——每一步都是一道关，过不了就拦下：

```
代码提交 → 编译 → 静态检查 → 单元测试 → 镜像构建 → 部署到测试 → 验收测试 → 发布
```

| 阶段 | 内容 | 失败处理 |
|:----|:------|:------|
| 编译 | make / mvn / go build | 阻断，通知开发者 |
| 静态检查 | SonarQube / Cppcheck / lint | 设置质量门禁阈值 |
| 单元测试 | GTest / pytest / JUnit | 必须 100% 通过 |
| 镜像构建 | docker build + push | 阻断 |
| 部署测试 | 部署到 test 环境 | 人工介入 |
| 验收测试 | E2E / 冒烟测试 | 阻断发布 |
| 发布 | 金丝雀 / 滚动更新 | 自动回滚或人工决策 |

设计时有两条原则值得记住：**越早的阶段越快**（编译/静态检查要秒级，不能拖到开发者不耐烦）；**越靠近发布的阶段越谨慎**（增加人工审批、灰度、回滚预案）。

## 3.3 多环境管理

为什么要分多个环境？因为**风险要分级消化**：谁也不想让一个未经验证的改动直接面对真实用户。

| 环境 | 用途 | 数据 | 稳定性要求 |
|:----|:------|:------|:---:|
| **dev** | 开发调试、快速验证 | 造数/脱敏 | 低 |
| **test** | 集成测试、功能验证 | 造数/脱敏 | 中 |
| **staging** | 预发布、生产镜像验证 | 脱敏生产数据 | 高 |
| **production** | 真实用户服务 | 真实数据 | 最高 |

代码是**逐级晋级**的——在 dev 通过、去 test 集成、到 staging 压测、最后上 prod：

```
dev（开发自测）→ 提交 MR 触发 CI
  ↓
test（集成测试 + 自动化回归）
  ↓ 通过 → 晋级
staging（预发布验证 + 压测）
  ↓ 通过 → 发布审批
production（金丝雀 → 全量）
```

### 环境一致性的三条铁律

- **代码一致**：**同一个镜像**贯穿所有环境——绝不"在每个环境重新构建一次"，否则测的不是同一个东西。
- **配置分离**：环境差异只体现在 ConfigMap / Secret，镜像不变。
- **基础设施一致**：集群配置、节点规格尽量对齐，减少"在测试好好的，上生产就崩"。

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1.2.3           # 所有环境同一镜像
        envFrom:
        - configMapRef:
            name: app-config-{{env}}   # 按环境注入不同配置
```

### 环境隔离

环境之间必须隔离，否则一个环境的操作会污染另一个：

| 维度 | 隔离方式 |
|:----|:------|
| 网络隔离 | 不同 Namespace / VPC / 网段 |
| 权限隔离 | RBAC：dev 只能操作 dev |
| 资源隔离 | 不同节点池 / 资源配额 |
| 数据隔离 | 独立数据库，禁止跨环境访问 |

### 多环境流水线

用 `when` 控制"什么分支部署到什么环境"，是最常见的多环境编排方式：

```groovy
pipeline {
    stages {
        stage('Build') { steps { sh 'docker build' } }
        stage('Deploy Dev') { steps { sh 'kubectl apply -f dev/' } }
        stage('Deploy Test') {
            when { branch 'develop' }
            steps { sh 'kubectl apply -f test/' }
        }
        stage('Deploy Staging') {
            when { branch 'release/*' }
            steps { sh 'kubectl apply -f staging/' }
        }
        stage('Deploy Prod') {
            when { branch 'main' }
            input { message '确认发布到生产？' }    // 人工审批闸门
            steps { sh 'kubectl apply -f prod/' }
        }
    }
}
```

## 3.4 发布策略

发布策略要回答的核心问题：**怎么把新版本交给用户，同时把风险降到最低**。不同策略在这条"风险-成本"轴上各有位置。

| 策略 | 原理 | 风险 | 回滚速度 | 资源成本 | 适用场景 |
|:----|:------|:---:|:---:|:---:|:------|
| **滚动更新** | 逐个替换实例，分批完成 | 低 | 快 | 低 | 无状态服务（大部分场景） |
| **蓝绿部署** | 新旧两套完整环境，切流量 | 极低 | 秒级 | 高（双倍） | 有状态服务、大版本变更 |
| **金丝雀发布** | 小比例流量先验证，逐步放量 | 中 | 快 | 中 | 需要真实流量验证 |
| **A/B 测试** | 按用户特征分流，对比效果 | 低 | 快 | 中 | 产品决策、功能验证 |
| **影子发布** | 镜像生产流量但不影响用户 | 零 | 无需回滚 | 中 | 新架构验证、压测 |

### 滚动更新

K8s 默认策略：**逐个替换**旧实例，用 `maxSurge`（允许超出多少）和 `maxUnavailable`（允许少多少）控制节奏。优点是不需要额外资源、平滑；缺点是新旧版本会**短暂共存**，对版本兼容性有要求。

```
v1  v1  v1  v1          v1  v2  v1  v1         v2  v2  v2  v2
[ ] [ ] [ ] [ ]   →   [ ] [ ] [ ] [ ]   →   [ ] [ ] [ ] [ ]
```

### 蓝绿部署

维护两套**完整环境**：老的 Blue 在跑，新的 Green 部署好待命，验证 OK 后**一次性把流量切过去**。回滚就是切回来，秒级。代价是**双倍资源**，而且要提前处理数据库这类有状态组件的兼容。

```
┌─────────────┐      切换 DNS/LB       ┌─────────────┐
│   🔵 Blue   │ ──────────────────→   │   🟢 Green  │
│   v1 运行中  │                        │   v2 就绪    │
└─────────────┘                        └─────────────┘
```

### 金丝雀发布

名字来自"矿工带金丝雀下井试毒"。做法是**先放一小撮真实流量给新版本**，观察指标没问题再逐步放量：

```
10% 用户 → v2（观察 15 分钟）→ ✅
50% 用户 → v2（观察 15 分钟）→ ✅
100% 用户 → v2
```

关键观察指标：**错误率、延迟 P99、业务指标**（如下单成功率）。工具常用 Istio / Linkerd（流量分割）、Argo Rollouts。

### 怎么选：决策树

```
需要按用户分流量验证产品效果？
  ├─ 是 → A/B 测试
  └─ 否 → 需要零风险切换？
            ├─ 是 → 蓝绿部署
            └─ 否 → 需要真实流量逐步验证？
                      ├─ 是 → 金丝雀发布
                      └─ 否 → 滚动更新（默认选择）
```

### 回滚策略

再完善的发布也要有退路：

| 场景 | 回滚方式 |
|:----|:------|
| 滚动更新 | `kubectl rollout undo` |
| 蓝绿部署 | LB 切回 Blue |
| 金丝雀 | 流量权重调回 0%，或 rollout undo |
| 数据库变更 | **反向迁移脚本（必须有！）** |

数据库变更是最容易被忽略的回滚难点——代码能秒回，但改过的表结构/数据得有**反向迁移**才能回去。

### 发布 Checklist

- [ ] Code Review 通过
- [ ] CI 流水线全部绿灯
- [ ] 监控仪表盘已配置
- [ ] 告警规则已检查
- [ ] 回滚方案已确认
- [ ] 数据库迁移已准备（含反向）
- [ ] 通知已发出

## 3.5 GitOps

GitOps 的核心思想：**把 Git 当作集群状态的"唯一事实源"**。集群里跑什么，全写在一份 Git 仓库里；有个 Agent（如 ArgoCD）盯着这份仓库，发现集群实际状态和 Git 里声明的期望状态不一致，就自动把它拉回来。

| 工具 | 特点 |
|:----|:------|
| **ArgoCD** | K8s 原生，Web UI 强大，自动同步 |
| **Flux CD** | 轻量，Helm 集成好，Git 原语操作 |

```
开发者 Push 代码 → CI 构建镜像 → 更新部署仓库的 YAML → ArgoCD 检测变更 → 自动同步到集群
```

**四大原则**：声明式（描述期望状态）、版本化且不可变（存 Git、可追溯）、拉取式自动同步（控制器主动拉）、持续调谐（自动纠偏，即 Drift Detection）。

GitOps 与传统 CI/CD 最大的区别在**推送方向**：

| 模式 | 机制 | 优点 | 缺点 |
|:----|:------|:------|:------|
| **Push**（传统） | CI 直接 `kubectl apply` | 简单直观 | CI 得持有集群凭证，安全面大 |
| **Pull**（GitOps） | 集群内 Agent 拉 Git | 凭证不出集群，可审计、可回滚 | 需额外部署控制器 |

**为什么 Pull 更安全？** 传统模式下，CI 系统必须拿到集群的管理员凭证才能部署——一旦 CI 被攻破，集群就裸奔了。GitOps 反过来：部署凭证只存在集群内部的 Agent 手里，CI 只负责往 Git 推代码，攻击面小得多。

## 3.6 Repo — 多仓库管理

一个大型嵌入式项目，往往拆成**几十甚至上百个 Git 仓库**（BSP 层、中间件层、各应用层各自独立）。一个个 `git clone`/`git pull` 显然不现实——**Repo** 就是 Google 为 Android/AOSP 开发的多仓库管理工具，专门解决这个问题。

它的核心是一份 **manifest（XML 清单）**：把所有仓库的地址、分支、路径写在一起，之后 `repo sync` 一键同步全部，`repo start` 在全部仓库创建同名分支，`repo upload` 批量提交评审。

| 概念 | 说明 |
|:----|:------|
| **manifest** | XML 清单，定义所有仓库的地址、分支、路径 |
| **default.xml** | 默认清单，描述整个项目的组成 |
| **repo sync** | 一键同步所有仓库到指定分支/标签 |
| **repo start** | 在所有相关仓库创建主题分支 |
| **repo upload** | 批量推送到 Gerrit 评审 |

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <remote name="origin" fetch="ssh://git@gerrit.example.com:29418/" />
  <default revision="main" remote="origin" sync-j="4" />   <!-- 并行 4 路同步 -->

  <project path="bsp/mcu-a"      name="platform/bsp/mcu-a"      revision="v2.1.0" />
  <project path="bsp/mcu-b"      name="platform/bsp/mcu-b"      revision="v2.1.0" />
  <project path="middleware/someip" name="middleware/someip" />
  <project path="apps/cluster"   name="apps/cluster" />
  <project path="apps/ivi"       name="apps/ivi" />
  <project path="tools/build"    name="tools/build" />
</manifest>
```

```bash
repo init -u ssh://git@gerrit/project/manifest.git -b main  # 初始化（拉取 manifest）
repo sync                                                    # 同步所有仓库
repo start feature-xxx --all                                 # 全部仓库开分支
repo status                                                  # 查看全部变更
repo upload                                                  # 批量提交 Gerrit 评审
repo forall -c 'git clean -fdx'                              # 对所有仓库执行命令
```

和 Git submodule 的区别：submodule 是"每个仓库各自记录子仓库的指针"，逐个更新很繁琐；Repo 是"用一份清单统一管理"——**适合"仓库多且需要整体协同"的场景**（这也是汽车/AOSP 类项目的标配）。

## 3.7 Gerrit — 代码评审系统

Gerrit 和 GitLab MR 最大的不同在**评审粒度**：GitLab 评审的是"一个分支/MR"，而 Gerrit 评审的是**单个 Commit**。每个 Commit 进 Gerrit 后是一个独立的"Change"，靠 `Change-Id` 跟踪。

| 维度 | Gerrit | GitLab MR |
|:----|:------|:----------|
| 评审粒度 | 单个 Commit | 整个分支/MR |
| 工作流 | Commit → Review → amend 重提 → +2 → Submit | Push 分支 → MR → Review → Merge |
| 权限模型 | 细粒度：+1 / +2 分权 | 相对粗粒度 |
| 适用场景 | 大型开源项目（AOSP）、汽车行业 | 通用企业 |
| 学习曲线 | 陡峭 | 平缓 |

Gerrit 的权限模型值得留意：**Code-Review +1** 表示"我看过，认可但不够格合并"，**+2** 才是"批准合并"；CI 回写的是 **Verified +1/-1**（验证通过与否）。两套独立的分数，把人评审和机器验证分开。

```
开发者 git commit → repo upload
    ↓
Gerrit 接收为一个 Change
    ↓
CI 触发验证（基于 refs/changes/xx/yy/zz 这个特殊 ref）
    ↓
CI 回写 Verified +1 / -1
    ↓
人工 Review（Code-Review +2）
    ↓
Submit → 合入目标分支
```

CI 通过 Jenkins 的 Gerrit Trigger 插件监听变更：

```groovy
pipeline {
    agent any
    triggers {
        gerrit(
            project: 'platform/bsp/mcu-a',
            branch: 'refs/heads/main',
            triggerOn: [
                patchsetCreated(approvalCategory: 'Code-Review', approvalValue: '+1')
            ]
        )
    }
    stages {
        stage('Verify') {
            steps {
                sh 'cmake -B build && cmake --build build'
                sh 'ctest --test-dir build --output-on-failure'
            }
        }
    }
}
```

## 3.8 KubeSphere — 容器平台

KubeSphere 是在原生 K8s 之上**加了一层"好用"**的平台。原生 K8s 功能强但门槛高（RBAC 要手写、监控日志要自己搭、流水线要自己装 Jenkins），KubeSphere 把这些打包成了开箱即用的一套。

| 模块 | 功能 |
|:----|:------|
| **多租户管理** | 企业空间 → 项目 → 资源三级隔离，比原生 RBAC 友好 |
| **DevOps 流水线** | 内置 Jenkins 引擎，可视化编排 |
| **灰度发布** | 蓝绿 / 金丝雀 / 流量镜像一键配置 |
| **微服务治理** | 基于 Istio / Spring Cloud 的服务网格 |
| **应用商店** | 基于 Helm 的应用模板市场 |
| **可观测性** | 内置 Prometheus + Grafana + EFK 日志 |

```
拉取代码 → 单元测试 → 代码分析 → 镜像构建 → 推送仓库 → 部署到开发环境 → 部署到生产
```

KubeSphere 把 Jenkins Pipeline 封装成可视化编排，支持凭证管理、构建节点池、流水线模板——**降低了维护 Jenkins 的成本**。

| 场景 | KubeSphere 的优势 |
|:----|:------|
| 团队权限管理 | 企业空间/项目/角色体系，无需手写 RBAC |
| 流水线 | 开箱即用的 DevOps 引擎 |
| 日志/监控 | 内置可观测性，省去自建 EFK/Prometheus |
| 应用发布 | 灰度/蓝绿/金丝雀一键配置 |

## 3.9 多技术栈统一流水线设计

现实中的团队往往要同时维护 C++ / Java / Go / Python 多种技术栈，如果每个栈一套流水线，维护成本会失控。统一流水线的思路是**抽公共、分差异**：

```
              ┌─────────────────────┐
              │    代码仓库层        │
              │ Git / Repo / Gerrit │
              └─────────┬───────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ C++ 构建 │  │ Java构建 │  │ Go/Py    │
  │ CMake    │  │ Maven    │  │ go build │
  │ Conan    │  │ Gradle   │  │ pip      │
  │ 交叉编译 │  │ Spring   │  │          │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │             │             │
       └─────────────┼─────────────┘
                     ▼
           ┌──────────────────┐
           │    质量门禁       │      ← 公共环节（统一）
           │ SonarQube/Trivy  │
           └────────┬─────────┘
                    ▼
           ┌──────────────────┐
           │  制品 & 部署      │      ← 公共环节（统一）
           │ Nexus + K8s      │
           └──────────────────┘
```

**公共环节**（拉代码、质量门禁、制品归档、部署）抽成 Shared Library，各语言栈只写自己的"构建 + 测试"差异部分——这样加一个新项目，只需填几个参数即可复用整条流水线。
