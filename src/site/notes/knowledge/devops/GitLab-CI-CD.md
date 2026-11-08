---
{"dg-publish":true,"permalink":"/knowledge/devops/GitLab-CI-CD/","title":"GitLab CI/CD","tags":["GitLab","GitLab-CI","CI-CD","Runner","DevOps","流水线"],"dg-note-properties":{"title":"GitLab CI/CD","date":"2026-11-06","tags":["GitLab","GitLab-CI","CI-CD","Runner","DevOps","流水线"]}}
---


# 🦊 GitLab CI/CD

> GitLab 不只是代码托管——它把 **仓库 + CI/CD + 制品库 + 安全扫描 + 环境管理** 做在一个平台里。
> 相比 Jenkins「自己拼装插件」的方式，GitLab CI 通过一个 `.gitlab-ci.yml` 就能跑通完整流水线。

返回 [[knowledge/devops/00-索引\|🧭 DevOps 知识体系]]

---

## 一、GitLab 是什么

GitLab 是**一体化 DevOps 平台**，一个安装包里有：

| 组件 | 作用 |
|:--|:--|
| **Git 仓库托管** | 代码托管、MR（Merge Request）、代码评审 |
| **GitLab CI/CD** | 流水线引擎（本文重点） |
| **Container Registry** | 内置镜像仓库 |
| **Package Registry** | npm / Maven / PyPI / NuGet 等制品库 |
| **Security** | SAST / DAST / 依赖扫描 / 密钥检测 |
| **Environments** | 环境管理与发布审批 |
| **Pages** | 静态站点托管 |

**版本区别**：

| 版本 | 说明 |
|:--|:--|
| **CE（社区版）** | 开源免费，功能已很完整 |
| **EE（企业版）** | 付费，多出合规审计、多级审批等企业特性 |
| **极狐 GitLab（JiHu）** | 国内独立发行版，基于 GitLab EE，境内有服务支持 |

> 国内企业多用极狐版；自建首选 CE，够用且成本低。

---

## 二、核心概念

在写第一行 `.gitlab-ci.yml` 之前，先把这几个词理顺——它们是最常见的混淆点。

```
Pipeline（流水线）
└── Stage（阶段）
    └── Job（作业）
        └── Runner（执行器）
```

| 概念 | 说明 |
|:--|:--|
| **Pipeline** | 一次完整的流水线运行，由一次 Push/MR/定时触发 |
| **Stage** | 阶段，**同 Stage 的 Job 并行**，不同 Stage **按顺序执行** |
| **Job** | 最小执行单元，一个 Job = 一个脚本任务 |
| **Runner** | 执行 Job 的机器/容器，由 GitLab 分配任务 |
| **Executor** | Runner 的运行方式（docker / shell / kubernetes / ssh） |
| **Artifact** | Job 产出的文件，可在后续 Stage 或下载页拿到 |
| **Cache** | 跨 Pipeline 复用的缓存（如 node_modules、maven 仓库） |

**Stage 与 Job 的关系**：

```
Pipeline
├── Stage: build
│   ├── Job: build-arm64
│   └── Job: build-x86     ← 并行
├── Stage: test
│   └── Job: unit-test     ← 等 build 全部完成才开始
└── Stage: deploy
    └── Job: deploy-prod   ← 手动触发（when: manual）
```

**Artifact 与 Cache 的区别**（高频面试题）：

| | Artifact | Cache |
|:--|:--|:--|
| 生命周期 | 只在本次 Pipeline 内 | 跨 Pipeline 持久 |
| 用途 | 把构建产物传给下个 Stage | 加速构建（依赖包） |
| 存储位置 | GitLab 服务器 | Runner 本地 / 对象存储 |
| 典型内容 | `.jar`、`.apk`、测试报告 | `node_modules`、`.m2` |

---

## 三、`.gitlab-ci.yml` 基础

文件放在**仓库根目录**，GitLab 检测到就自动启用流水线。

### 3.1 最小示例

```yaml
stages:
  - build
  - test
  - deploy

build-job:
  stage: build
  script:
    - echo "编译中..."
    - make build
  artifacts:
    paths:
      - build/

test-job:
  stage: test
  script:
    - make test

deploy-job:
  stage: deploy
  script:
    - ./deploy.sh
  when: manual          # 手动点击才执行
```

### 3.2 常用关键字

| 关键字 | 作用 |
|:--|:--|
| `stages` | 定义阶段顺序 |
| `script` | Job 执行的命令（必须有） |
| `before_script` / `after_script` | 全局或 Job 级前后置脚本 |
| `image` | 指定 Docker 镜像作为执行环境 |
| `services` | 声明依赖服务（如 MySQL、Redis 容器） |
| `variables` | 定义环境变量 |
| `rules` | **条件控制**（何时执行/不执行） |
| `only` / `except` | 旧版条件语法，**建议用 `rules`** |
| `artifacts` | 产物归档 |
| `cache` | 缓存 |
| `needs` | 跨 Stage 依赖（DAG） |
| `tags` | 指定 Runner 标签 |
| `when` | `on_success` / `on_failure` / `always` / `manual` / `never` |
| `allow_failure` | 允许失败但不阻塞流水线 |
| `timeout` | Job 超时时间 |

### 3.3 `script` 的写法要点

```yaml
job:
  script:
    # 多行写法（推荐，每行是独立的 shell 命令）
    - echo "step 1"
    - ls -la

    # 用 | 写多行脚本块（共享同一 shell 会话）
    - |
      echo "step 2"
      cd build
      make
```

> ⚠️ **坑**：`-` 后面的每一条是**独立 shell**。想保持 `cd` 效果，用 `|` 块或者 `cd xxx && make`。

---

## 四、变量管理

### 4.1 三种变量来源

| 来源 | 优先级 |
|:--|:--|
| `.gitlab-ci.yml` 里的 `variables` | 最低 |
| 项目/群组/实例级的 CI/CD Variables（界面配置） | 中 |
| 手动触发的 pipeline 变量 | 最高 |

### 4.2 内置变量（常用）

| 变量 | 含义 |
|:--|:--|
| `CI_COMMIT_SHA` | 本次提交的完整 SHA |
| `CI_COMMIT_SHORT_SHA` | 短 SHA（8 位），**打镜像 tag 常用** |
| `CI_COMMIT_BRANCH` | 分支名 |
| `CI_COMMIT_TAG` | Tag 名（Tag 触发时才有） |
| `CI_PROJECT_PATH` | 项目路径（如 `group/project`） |
| `CI_PIPELINE_ID` | 流水线 ID |
| `CI_JOB_ID` | Job ID |
| `CI_REGISTRY_IMAGE` | 本项目容器镜像的完整地址 |
| `CI_MERGE_REQUEST_IID` | MR 编号（MR 触发时才有） |

### 4.3 保护变量

敏感信息（密码、Token）在 **Settings → CI/CD → Variables** 配置：

| 选项 | 作用 |
|:--|:--|
| **Protected** | 只在受保护分支/Tag 可用 |
| **Masked** | 日志里自动脱敏 |
| **Expanded** | 是否展开变量嵌套 |

```yaml
deploy:
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
```

> ⚠️ **绝不**把密码写进 `.gitlab-ci.yml`——文件会进 Git 历史，泄露无法收回。

---

## 五、`rules`：条件控制的核心

`rules` 决定「这个 Job 什么时候跑」，是配置文件里最容易写错的部分。

### 5.1 基础用法

```yaml
build:
  script: make build
  rules:
    # 1) 只在 MR 时跑
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: always

    # 2) 只在 main 分支跑
    - if: '$CI_COMMIT_BRANCH == "main"'

    # 3) 有 tag 时跑
    - if: '$CI_COMMIT_TAG'

    # 4) 其他情况不跑
    - when: never
```

> **规则从上往下匹配，命中第一条就停止**。所以 `when: never` 一定要放最后。

### 5.2 典型场景

```yaml
# 只在 MR 且变更了 src/ 目录时跑
test:
  script: make test
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - src/**/*

# 定时任务才跑
nightly:
  script: ./run-e2e.sh
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
```

### 5.3 `rules` vs `only/except`

| | `rules` | `only/except` |
|:--|:--|:--|
| 表达能力 | 强（if / changes / exists） | 弱 |
| 可读性 | 好 | 易混淆 |
| 官方态度 | **推荐** | 保留兼容，不推荐新项目使用 |

---

## 六、`needs`：DAG 加速

默认情况下，Stage 是按顺序执行的：`build` 全部完成才能开始 `test`。但很多时候并不需要这样等。

### 6.1 问题场景

```
Stage: build
├── build-slow   (10 分钟)
└── build-fast   (1 分钟)

Stage: test
└── test-fast    (只依赖 build-fast)
```

不用 `needs` 时：`test-fast` 要等 `build-slow` 跑完，白等 9 分钟。

### 6.2 用 `needs` 打破顺序

```yaml
build-slow:
  stage: build
  script: make heavy-build

build-fast:
  stage: build
  script: make quick-build

test-fast:
  stage: test
  needs: ["build-fast"]     # 只等 build-fast，不等整个 build stage
  script: make test
```

### 6.3 `needs` 的额外价值

- **产物自动传递**：`needs` 会下载依赖 Job 的 artifacts
- **跨 Stage 依赖**：可以依赖后续 Stage 的 Job，实现灵活编排
- **大幅提速**：大项目从串行 40 分钟降到并行 15 分钟很常见

```yaml
# 需要并行下载多个 Job 的产物
deploy:
  needs:
    - job: build-arm64
      artifacts: true
    - job: build-x86
      artifacts: true
```

---

## 七、Runner 部署

Runner 是真正干活的部分。没有可用 Runner，Job 会一直 `pending`。

### 7.1 三种常见 Executor

| Executor | 说明 | 适用 |
|:--|:--|:--|
| **shell** | 直接跑在 Runner 宿主机上 | 简单、但环境易污染 |
| **docker** | 每个 Job 在独立容器里跑 | ✅ **最常用**，环境隔离 |
| **kubernetes** | 每个 Job 起一个 Pod | ✅ 云原生环境首选 |

### 7.2 安装（Docker Executor）

```bash
# 1) 在 GitLab 拿注册 Token
#    Settings → CI/CD → Runners → New project runner

# 2) 起 Runner 容器
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest

# 3) 注册
docker exec -it gitlab-runner gitlab-runner register \
  --url https://gitlab.example.com/ \
  --token <YOUR_TOKEN> \
  --executor docker \
  --docker-image alpine:latest
```

### 7.3 Kubernetes Executor（config.toml）

```toml
[[runners]]
  name = "k8s-runner"
  url = "https://gitlab.example.com/"
  token = "xxx"
  executor = "kubernetes"
  [runners.kubernetes]
    namespace = "gitlab-runner"
    image = "alpine:latest"
    cpu_request = "500m"
    memory_request = "512Mi"
    service_account = "gitlab-runner"
    # 构建镜像用的 dind
    [[runners.kubernetes.services]]
      name = "docker:24-dind"
      alias = "docker"
```

### 7.4 Runner 的三种类型

| 类型 | 说明 |
|:--|:--|
| **Shared Runner** | 实例级共享，所有项目可用 |
| **Group Runner** | 群组级，组内项目可用 |
| **Specific Runner** | 项目专属，配置最灵活 |

### 7.5 ⚠️ 常见坑

| 坑 | 现象 | 解决 |
|:--|:--|:--|
| **Runner 不打 tag** | Job 永远 pending | Job 里写 `tags:`，或 Runner 设 `run_untagged = true` |
| **旧 pending 不自动接管** | 注册新 Runner 后老 Job 还卡着 | 手动 retry 那条流水线 |
| **entrypoint 覆盖** | 容器启动命令被镜像 ENTRYPOINT 顶掉 | `image.entrypoint: [""]` |
| **TOML 键里有 `/`** | 解析报错 | 键名加引号：`"docker.io"` |
| **`empty_dir` / `pvc` 写成对象** | 校验失败 | 它们是**数组** |
| **`FF_USE_FASTZIP` 重复定义** | 覆盖冲突 | 只在全局设一次 |

---

## 八、制品与缓存实战

### 8.1 缓存依赖（加速构建）

```yaml
build:
  cache:
    key:
      files:
        - package-lock.json      # lock 文件变了才重建缓存
    paths:
      - node_modules/
  script:
    - npm ci
    - npm run build
```

### 8.2 归档产物

```yaml
build:
  artifacts:
    name: "app-$CI_COMMIT_SHORT_SHA"
    paths:
      - dist/
    expire_in: 1 week            # 过期自动清理
```

### 8.3 测试报告（原生支持，无需插件）

```yaml
test:
  script:
    - pytest --junitxml=report.xml
  artifacts:
    when: always                 # 测试失败也要报告
    reports:
      junit: report.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
```

> 这比 Jenkins 强很多——Jenkins 要装 JUnit 插件，GitLab 原生支持 `reports:junit`。

---

## 九、镜像构建与推送

用 GitLab 内置的 Container Registry，无需额外配置凭证：

```yaml
build-image:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    - TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}"
    - docker build -t "$CI_REGISTRY_IMAGE:$TAG" .
    - docker push "$CI_REGISTRY_IMAGE:$TAG"
  rules:
    - if: '$CI_COMMIT_TAG'
    - if: '$CI_COMMIT_BRANCH == "main"'
```

**要点**：
- `dind`（Docker in Docker）是共享 Docker 最常见的做法
- tag 用 `CI_COMMIT_SHORT_SHA` 而不是 `latest`——**镜像不可变**才可回滚
- 生产环境建议用 **Kaniko**（无需 privileged）或 **BuildKit**

---

## 十、多环境与发布

### 10.1 Environments

```yaml
deploy-staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - ./deploy.sh staging

deploy-prod:
  stage: deploy
  environment:
    name: production
    url: https://example.com
  when: manual
  script:
    - ./deploy.sh prod
  rules:
    - if: '$CI_COMMIT_TAG'
```

定义 `environment` 后，GitLab 页面会出现 **Environments** 面板：能看到当前部署了哪个版本，还能**一键回滚（Re-deploy）**。

### 10.2 受保护环境

**Settings → CI/CD → Protected environments**：限制谁能部署到生产环境。

### 10.3 发布审批

EE 版支持 **Deployment Approvals**：生产部署前必须有指定人员批准。

---

## 十一、跨项目/多流水线

### 11.1 `include`：模板复用

```yaml
include:
  # 同仓库
  - local: '/templates/.build-template.yml'

  # 其他项目
  - project: 'devops/ci-templates'
    ref: main
    file: '/templates/java.yml'

  # 远程 URL
  - remote: 'https://example.com/ci/standard.yml'
```

### 11.2 `extends`：继承

```yaml
.build-base:
  image: maven:3.9
  before_script:
    - mvn -v

build-app:
  extends: .build-base       # 继承上述配置
  script:
    - mvn clean package
```

### 11.3 `trigger`：触发下游流水线

```yaml
trigger-downstream:
  trigger:
    project: other-group/other-project
    branch: main
    strategy: depend          # 等下游完成
```

---

## 十二、Jenkins 与 GitLab CI 对比

| 维度 | Jenkins | GitLab CI |
|:--|:--|:--|
| **配置方式** | Jenkinsfile（Groovy） | `.gitlab-ci.yml`（YAML） |
| **配置格式** | 图灵完备，灵活但复杂 | 声明式，简单但表达力有限 |
| **复用机制** | Shared Library | `include` + `extends` + `!reference` |
| **并行** | 同 stage 并行 | 同 stage 并行 + **`needs` 原生 DAG** |
| **测试报告** | 装插件 | **原生** `reports:junit` |
| **MR 门禁** | 插件 + 回写状态 | **内置**，MR 页直接显示 |
| **增量构建** | 需 `discoverReferenceBuild` | `CI_MERGE_REQUEST_DIFF_BASE_SHA` 直接给 |
| **维护成本** | 高（插件升级、Master 维护） | 低（与仓库一体） |
| **灵活性** | **极高**（Groovy 随便写） | 中等 |
| **适用** | 复杂流程、嵌入式/汽车 | 中小团队、云原生 |

**选型建议**：
- 源码在 GitLab + 流程标准 → **GitLab CI**
- 需要复杂定制（交叉编译、Windows 工控机、License 绑定）→ **Jenkins**
- 两者共存：GitLab CI 做常规流水线，Jenkins 做特殊构建

---

## 十三、常见坑

| 坑 | 现象 | 解决 |
|:--|:--|:--|
| Job 一直 `pending` | Runner 标签不匹配 | 检查 `tags:` 和 Runner 注册时的标签 |
| `cd` 不生效 | 每条 `script` 是独立 shell | 用 `\|` 块或 `cd x && cmd` |
| 变量没展开 | 单引号不插值 | YAML 里用双引号，shell 里注意引号层级 |
| MR 流水线跑两遍 | 分支推送 + MR 事件都触发 | `rules` 里明确区分 `$CI_PIPELINE_SOURCE` |
| `rules` 顺序写错 | 该跑的没跑 | 记住**从上往下匹配，命中即停** |
| cache 不命中 | key 太宽泛 | 用 `key.files` 绑定 lock 文件 |
| artifacts 太大超时 | 打包了 `node_modules` | 只归档必要产物，用 `expire_in` |
| dind 权限不足 | `docker: command not found` | 检查 `services` 和 `DOCKER_TLS_CERTDIR` |
| 受保护变量在 MR 里为空 | MR 来自 fork | fork 的 MR 拿不到受保护变量，属安全设计 |

---

## 十四、一句话总结

> **GitLab CI = 仓库 + `.gitlab-ci.yml` + Runner**。
> 三个阶段：定义 Stage → 写 Job 脚本 → 配 `rules` 控制时机；需要加速就上 `needs`（DAG）。

---

## 相关文档

- [[knowledge/devops/jenkins/00-索引\|🧩 Jenkins]] — 另一套 CI 平台（功能模块详解）
- [[knowledge/devops/argocd/00-索引\|🚢 ArgoCD]] — GitOps 持续部署
- `knowledge/devops/practices/03-CI工程化实战.md` — GitLab CI 实战与双平台对比
- `knowledge/devops/practices/01-Jenkins部署与迁移实战.md` — Jenkins 部署实操
