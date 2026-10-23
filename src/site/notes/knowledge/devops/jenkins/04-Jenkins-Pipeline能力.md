---
{"dg-publish":true,"permalink":"/knowledge/devops/jenkins/04-Jenkins-Pipeline能力/","tags":["Jenkins","Pipeline","Jenkinsfile","SharedLibrary","DevOps"],"dg-note-properties":{"date":"2026-09-16","tags":["Jenkins","Pipeline","Jenkinsfile","SharedLibrary","DevOps"]}}
---


# 🧩 Jenkins Pipeline 能力

> Jenkins 功能模块全景 · 第四章。Pipeline 是 Jenkins 最核心的能力——把 CI/CD 流程写成代码。

---

## 一、Jenkinsfile

定义流水线的**代码文件**，通常放在仓库根目录，随代码进 Git 评审。

- **Declarative**（声明式）：`pipeline { agent / stages / post }` 固定骨架，推荐
- **Scripted**（脚本式）：`node { }` 纯 Groovy，灵活但难维护

**两种风格骨架对比**：

```groovy
// 声明式
pipeline {
    agent any
    stages {
        stage('Build') {
            steps { sh 'make' }
        }
    }
    post {
        always  { echo '清理' }
        success { echo '成功' }
        failure { echo '失败' }
    }
}

// 脚本式
node {
    stage('Build') {
        sh 'make'
    }
}
```

## 二、Stage / Step（阶段与步骤）

- **Stage**：逻辑阶段（如 构建 → 测试 → 部署），在 UI 上可视化展示
- **Step**：原子操作（`sh`、`git`、`archiveArtifacts`、`junit`…），Pipeline 的最小执行单元
- **`post`**：阶段后置条件（`always` / `success` / `failure` / `unstable` / `changed`）

## 三、Shared Library（共享库）⭐

**跨仓库复用流水线逻辑**的关键机制：

```
(根目录)/vars/         全局变量/函数（如 deploy.groovy → 可直接 @Library 调用）
        /src/          标准 Groovy 类
        /resources/    非 Groovy 资源（模板、配置）
```

**用法**：
```groovy
@Library('my-shared-lib') _
pipeline {
    agent any
    stages {
        stage('Build') { steps { myBuildStep('参数') } }
    }
}
```

**价值**：把复杂的流水线逻辑收口到一处，各仓库的 Jenkinsfile 保持"瘦"。**车载项目六仓共用一套 Shared Library 就是这个思路**。

## 四、并行执行

```groovy
// 并行 stage
parallel {
    stage('测试A') { steps { sh './test-a.sh' } }
    stage('测试B') { steps { sh './test-b.sh' } }
}

// 矩阵并行
matrix {
    axes {
        axis { name 'PLATFORM'; values 'linux', 'windows' }
    }
    stages { stage('Build') { steps { sh 'build' } } }
}
```

- **减少总时长**：慢活/快活拆开并行
- **注意**：并行会同时占多个 Executor，Agent 池要够

## 五、Agent 指令（在哪跑）

```groovy
agent any                                  // 任意可用节点
agent { label 'linux' }                    // 指定标签
agent { docker { image 'maven:3.9' } }     // 容器内执行
agent { kubernetes { yaml '''...''' } }    // K8s 动态 Pod
agent none                                 // 不分配，stage 内各自指定
```

- **`kubernetes` agent** = K8s 动态 Agent，按需拉 Pod、用完销毁（云原生首选）
- **`docker` agent** = 环境隔离，构建环境可复现
- Stage 级 agent 覆盖全局 agent，实现"不同阶段在不同环境跑"

## 六、超时 / 重试 / 护栏

```groovy
options {
    timeout(time: 30, unit: 'MINUTES')   // 全局超时
    retry(3)                             // 失败重试
    timestamps()                         // 日志加时间戳
    skipDefaultCheckout()                // 不自动 checkout
}
```

- `timeout` 防止挂死（**构建僵尸的常见解药**）
- `retry` 应对网络抖动类偶发失败
- `catchError` / `unstable` 可让阶段失败但不中断整体

## 七、Blue Ocean（可视化 UI）

Pipeline 的现代化图形界面：

- 流水线可视化（节点连线）
- 分支/PR 视图
- 更直观的日志与阶段耗时

> 注：Blue Ocean 插件已进入维护状态，但仍是理解 Pipeline 结构的好工具。

## 八、Pipeline 结构总览

```
Jenkinsfile
├── pipeline { }
│   ├── agent        ← 在哪跑
│   ├── options      ← 超时/重试/时间戳
│   ├── environment  ← 环境变量
│   ├── triggers     ← 触发方式
│   ├── parameters   ← 参数化
│   ├── stages
│   │   ├── stage 1 → steps / parallel / matrix
│   │   └── stage 2 → ...
│   └── post         ← always/success/failure/unstable/changed
└── @Library('shared-lib')  ← 共享库复用
```

> 💡 **实战三条**：① 能进 Shared Library 的逻辑别复制到各仓库；② 重活（扫描/Fuzz）只触发不硬算，算力放常驻服务；③ 用 `timeout` 兜住所有可能挂死的 stage。

---

> 🦐 虾管家 · 2026-09-16
