---
{"dg-publish":true,"permalink":"/knowledge/devops-general/DevOps-3-制品与配置管理/","tags":["DevOps","制品仓库","配置管理","Nexus","Harbor","Git"],"dg-note-properties":{"date":"2026-07-19","tags":["DevOps","制品仓库","配置管理","Nexus","Harbor","Git"]}}
---


# 三、制品与配置管理

## 3.1 代码管理

### Git 分支策略

| 策略 | 特点 | 适用 |
|:----|:------|:------|
| **Git Flow** | main / develop / feature / release / hotfix | 有版本发布周期的传统项目 |
| **Trunk-Based** | 全部提交到 main，短分支快速合并 | 持续部署、微服务 |
| **GitHub Flow** | main + feature 分支，PR 合并即上线 | 简单项目、SaaS |

### Code Review 流程

```
开发者创建 Feature 分支 → 提交 MR/PR
    → CI 自动跑（编译 + 测试 + 扫描）
    → 至少 1 人 Review + Approve
    → 合并到目标分支
    → 自动部署
```

## 3.2 制品仓库

### 制品类型与对应仓库

| 制品类型 | 仓库 | 说明 |
|:----|:------|:------|
| **Docker 镜像** | Harbor / Docker Hub | 容器镜像专用，支持安全扫描、镜像复制 |
| **Java jar/war** | Nexus / JFrog Artifactory | Maven/Gradle 依赖 + 构建产物 |
| **npm / pip / Go mod** | Nexus / Artifactory | 前端/脚本语言依赖代理缓存 |
| **通用二进制** | Nexus Raw / Artifactory Generic | .bin / .elf / .hex / .zip / .tar.gz |
| **Helm Chart** | Harbor / Chart Museum | K8s 应用包管理 |

### Harbor — Docker 镜像仓库

| 功能 | 说明 |
|:----|:------|
| **镜像管理** | Push / Pull / Tag / 删除 / 保留策略 |
| **安全扫描** | Trivy 集成，自动扫描镜像漏洞 |
| **访问控制** | 项目级 RBAC，LDAP/OIDC 认证 |
| **镜像复制** | 跨数据中心/跨云镜像同步 |
| **垃圾回收** | 自动清理未引用的镜像层 |

```bash
# 推送镜像
docker build -t myapp:v1.2.3 .
docker tag myapp:v1.2.3 harbor.example.com/prod/myapp:v1.2.3
docker push harbor.example.com/prod/myapp:v1.2.3
```

**镜像保留策略：** 保留最近 30 天 + 最近 10 版本，生产标签永久保留。

### Nexus — 通用制品仓库

| 类型 | 用途 |
|:----|:------|
| **Proxy** | 代理远程仓库（Maven Central、npm registry、PyPI），加速 + 离线可用 |
| **Hosted** | 私有制品托管 |
| **Group** | 聚合多个仓库为一个 URL，统一入口 |

```bash
# Maven 发布到 Nexus
mvn deploy
```

### 版本管理

```
v<MAJOR>.<MINOR>.<PATCH>[-<PRE-RELEASE>]

v1.2.3          → 正式版本
v1.2.3-rc1      → 候选版本
v1.2.3-SNAPSHOT → 开发快照
```

| | Release | Snapshot |
|:----|:---:|:---:|
| 版本 | 固定，不可变 | 可变，可覆盖 |
| 仓库 | releases | snapshots |
| 场景 | 正式上线 | 开发联调 |

### 制品元数据

```
制品名: myapp-v1.2.3.jar
Git Commit:  a1b2c3d
CI Job:      build #1234
构建时间:    2026-07-19 10:00:00
签名:        myapp-v1.2.3.jar.sha256
```

### 制品安全

| 措施 | 说明 |
|:----|:------|
| **访问控制** | 读/写分离，CI 写、部署读 |
| **镜像扫描** | 推送时自动扫描，高危镜像阻止部署 |
| **签名校验** | 制品签名 + 部署时验证，防篡改 |
| **网络隔离** | 制品仓库不直接暴露公网 |
| **审计日志** | 记录所有 Pull/Push 操作 |

### CI/CD 集成

```groovy
pipeline {
    environment {
        HARBOR = 'harbor.example.com'
        NEXUS = 'nexus.example.com'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
                sh 'docker build -t ${HARBOR}/prod/myapp:${BUILD_TAG} .'
            }
        }
        stage('Push') {
            steps {
                sh 'mvn deploy'
                sh "docker push ${HARBOR}/prod/myapp:${BUILD_TAG}"
            }
        }
        stage('Deploy') {
            steps {
                sh "kubectl set image deployment/myapp myapp=${HARBOR}/prod/myapp:${BUILD_TAG}"
            }
        }
    }
}
```

## 3.3 配置管理

| 工具 | 特点 |
|:----|:------|
| **ConfigMap / Secret** | K8s 原生，适合容器化环境 |
| **Apollo / Nacos** | 配置中心，支持动态刷新、灰度、权限管理 |
| **Vault** | 密钥管理，动态生成数据库凭证、加密存储 |

配置原则：
- 敏感信息用 Secret，不要硬编码在 ConfigMap 或代码里
- 按环境分层：base → dev → staging → prod
- 配置变更要有审计记录

## 3.4 项目管理

| 工具 | 用途 |
|:----|:------|
| **Jira** | 需求、Bug、版本规划 |
| **Confluence** | 技术文档、设计决策 |
| **飞书文档** | 团队协作、轻量文档 |
