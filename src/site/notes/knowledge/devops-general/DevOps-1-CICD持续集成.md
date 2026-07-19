---
{"dg-publish":true,"permalink":"/knowledge/devops-general/DevOps-1-CICD持续集成/","tags":["DevOps","CI/CD","Pipeline","Jenkins","GitLab","发布策略","多环境"],"dg-note-properties":{"date":"2026-07-19","tags":["DevOps","CI/CD","Pipeline","Jenkins","GitLab","发布策略","多环境"]}}
---


# 一、CI/CD 持续集成交付

## 1.1 流水线平台

| 平台 | 特点 | 适用场景 |
|:----|:------|:------|
| **Jenkins** | 最灵活，插件生态丰富，Pipeline as Code (Groovy) | 自建平台、复杂流程、嵌入式/汽车行业 |
| **GitLab CI** | 与 GitLab 深度集成，.gitlab-ci.yml 声明式配置 | 源码在 GitLab、中小团队 |
| **GitHub Actions** | GitHub 原生，Marketplace 生态好 | 开源项目、GitHub 为主的技术栈 |

### Jenkins Pipeline 示例

```groovy
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

### GitLab CI 示例

```yaml
stages:
  - build
  - test
  - deploy

build:
  stage: build
  script: make
  artifacts:
    paths: [build/]

test:
  stage: test
  script: ./run_tests.sh

deploy:
  stage: deploy
  script: kubectl apply -f deploy/
  only: [main]
```

## 1.2 流水线设计

标准 CI/CD 流水线阶段：

```
代码提交 → 编译 → 静态检查 → 单元测试 → 镜像构建 → 部署到测试 → 验收测试 → 发布
```

| 阶段 | 内容 | 失败处理 |
|:----|:------|:------|
| 编译 | make / mvn / go build | 阻断，通知开发者 |
| 静态检查 | SonarQube / Cppcheck / lint | 设置质量门禁阈值 |
| 单元测试 | GTest / pytest / JUnit | 必须 100% 通过 |
| 镜像构建 | docker build + push | 阻断 |
| 部署测试 | kubectl apply 到 test 环境 | 人工介入 |
| 验收测试 | E2E / 冒烟测试 | 阻断发布 |
| 发布 | 金丝雀 / 滚动更新 | 自动回滚或人工决策 |

## 1.3 多环境管理

### 环境层级

| 环境 | 用途 | 数据 | 稳定性要求 |
|:----|:------|:------|:---:|
| **dev** | 开发调试、快速验证 | 造数/脱敏 | 低 |
| **test** | 集成测试、功能验证 | 造数/脱敏 | 中 |
| **staging** | 预发布、生产环境镜像验证 | 脱敏生产数据 | 高 |
| **production** | 真实用户服务 | 真实数据 | 最高 |

### 环境晋升流程

```
dev（开发自测）
  ↓ 提交 MR → CI 自动跑
test（集成测试 + 自动化回归）
  ↓ 测试通过 → 手动/自动晋级
staging（预发布验证 + 压测）
  ↓ 验证通过 → 发布审批
production（金丝雀 → 全量）
```

### 环境一致性

- **代码一致**：同一镜像贯穿所有环境（不要在每个环境重新构建）
- **配置分离**：ConfigMap / Secret 按环境隔离，镜像不变
- **基础设施一致**：K8s 集群配置、节点规格尽量对齐，减少"环境不一致"导致的 Bug

```yaml
# K8s 配置示例：同一镜像，不同配置
# dev: configmap-dev.yaml / staging: configmap-staging.yaml / prod: configmap-prod.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1.2.3          # 所有环境同一镜像
        envFrom:
        - configMapRef:
            name: app-config-{{env}}  # 按环境注入不同配置
```

### 环境隔离

| 维度 | 隔离方式 |
|:----|:------|
| **网络隔离** | 不同 Namespace / VPC / 网段 |
| **权限隔离** | RBAC：dev 只能操作 dev 环境 |
| **资源隔离** | 不同节点池 / 资源配额 |
| **数据隔离** | 独立数据库实例，禁止跨环境访问 |

### 多环境流水线

```groovy
// Jenkins Pipeline 多环境示例
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
            input { message '确认发布到生产？' }
            steps { sh 'kubectl apply -f prod/' }
        }
    }
}
```

## 1.4 发布策略

### 策略对比

| 策略 | 原理 | 风险 | 回滚速度 | 适用场景 |
|:----|:------|:---:|:---:|:------|
| **滚动更新** | 逐个替换实例，分批完成 | 低 | 快 | 无状态服务（大部分场景） |
| **蓝绿部署** | 新旧两套完整环境，切流量 | 极低 | 秒级 | 有状态服务、大版本变更 |
| **金丝雀发布** | 小比例流量先验证，逐步放量 | 中 | 快 | 需要真实流量验证 |
| **A/B 测试** | 按用户特征分流，对比效果 | 低 | 快 | 产品决策、功能验证 |
| **影子发布** | 镜像生产流量但不影响用户 | 零 | 无需回滚 | 新架构验证、压测 |

### 滚动更新

K8s 默认策略，`maxSurge` + `maxUnavailable` 控制节奏。

```
v1  v1  v1  v1           v1  v2  v1  v1
[ ] [ ] [ ] [ ]  →  [ ] [ ] [ ] [ ]  →  [ ] [ ] [ ] [ ]
```

### 蓝绿部署

```
┌─────────────┐      切换DNS/LB       ┌─────────────┐
│   🔵 Blue   │ ──────────────────→  │   🟢 Green  │
│   v1 运行中  │                      │   v2 就绪    │
└─────────────┘                      └─────────────┘
```

注意：双倍资源消耗；数据库兼容性需提前处理。

### 金丝雀发布

```
10% 用户 → v2（观察 15 分钟）→ ✅
50% 用户 → v2（观察 15 分钟）→ ✅
100% 用户 → v2
```

关键指标：错误率、延迟 P99、业务指标（下单成功率等）。

工具：Istio / Linkerd（流量分割）、Argo Rollouts、Spinnaker。

### 选型决策树

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

| 场景 | 回滚方式 |
|:----|:------|
| 滚动更新 | `kubectl rollout undo` |
| 蓝绿部署 | LB 切回 Blue 环境 |
| 金丝雀 | 调整流量权重回 0%，或 rollout undo |
| 数据库变更 | 反向迁移脚本（必须有！） |

### 发布 Checklist

- [ ] Code Review 通过
- [ ] CI 流水线全部绿灯
- [ ] 监控仪表盘已配置
- [ ] 告警规则已检查
- [ ] 回滚方案已确认
- [ ] 数据库迁移已准备（含反向迁移）
- [ ] 通知已发出

## 1.5 GitOps

以 Git 为单一事实源驱动部署。

| 工具 | 特点 |
|:----|:------|
| **ArgoCD** | K8s 原生，Web UI 强大，自动同步 |
| **Flux CD** | 轻量，Helm 集成好，Git 原语操作 |

```
开发者 Push 代码 → CI 构建镜像 → 更新部署仓库的 YAML → ArgoCD 检测变更 → 自动同步到集群
```
