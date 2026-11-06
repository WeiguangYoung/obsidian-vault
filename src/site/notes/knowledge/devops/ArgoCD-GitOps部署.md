---
{"dg-publish":true,"permalink":"/knowledge/devops/ArgoCD-GitOps部署/","title":"ArgoCD","tags":["ArgoCD","GitOps","Kubernetes","持续部署","CD","声明式","同步"],"dg-note-properties":{"title":"ArgoCD","date":"2026-11-06","tags":["ArgoCD","GitOps","Kubernetes","持续部署","CD","声明式","同步"]}}
---


# 🚢 ArgoCD

> GitOps 的落地工具：**把 Git 仓库当作集群的唯一事实源**，ArgoCD 在集群里盯着这份仓库，发现实际状态和期望状态不一致就自动拉回来。
> 和传统「CI 直接 `kubectl apply`」相比，**部署凭证不出集群**，攻击面小得多。

返回 [[knowledge/devops/README\|🧭 DevOps 知识体系]]

---

## 一、GitOps 是什么

传统 CD 是 **Push 模式**：CI 系统拿着集群凭证，构建完直接推上去。

GitOps 是 **Pull 模式**：集群里跑一个 Agent，主动去 Git 拉期望状态。

| 模式 | 机制 | 优点 | 缺点 |
|:--|:--|:--|:--|
| **Push**（传统） | CI 直接 `kubectl apply` | 简单直观 | CI 得持有集群凭证，一旦被攻破集群裸奔 |
| **Pull**（GitOps） | 集群内 Agent 拉 Git | 凭证不出集群、可审计、可回滚 | 需额外部署控制器 |

**核心流程**：

```
开发者改代码 → CI 构建镜像 → CI 更新「部署仓库」的 YAML（改 image tag）
                                        ↓
                              Git 仓库（唯一事实源）
                                        ↓
                        ArgoCD 检测到差异 → 自动同步到集群
```

**四大原则**：

| 原则 | 含义 |
|:--|:--|
| **声明式** | 描述「期望状态」，不是「执行步骤」 |
| **版本化且不可变** | 全部存 Git，可追溯、可回滚 |
| **自动拉取** | 控制器主动拉，不靠外部推 |
| **持续调谐** | 实际状态偏离就自动纠偏（Drift Detection） |

> **一句话**：`git push` 就是部署动作。

---

## 二、ArgoCD 是什么

ArgoCD 是 CNCF 毕业的 GitOps 工具，专为 Kubernetes 设计。

| 特性 | 说明 |
|:--|:--|
| **声明式部署** | 从 Git 读取 K8s manifest / Helm / Kustomize |
| **Web UI** | 可视化资源树、同步状态、差异对比 |
| **自动同步** | 可配自动同步 + 自动纠正偏差 |
| **多集群** | 一个实例管多个集群 |
| **RBAC** | 细粒度权限控制 |
| **SSO** | 支持 OIDC / LDAP / GitHub / GitLab 登录 |

**和 Flux CD 的区别**：

| | ArgoCD | Flux CD |
|:--|:--|:--|
| UI | ✅ 自带，很强 | ❌ 需另配 |
| 上手难度 | 简单 | 稍复杂 |
| 资源占用 | 稍重 | 轻量 |
| Helm 集成 | 好 | 很好 |

> 有可视化需求、团队协作多 → ArgoCD；纯 CLI/轻量 → Flux。

---

## 三、安装

### 3.1 快速安装

```bash
# 建命名空间并安装
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 等待就绪
kubectl -n argocd wait --for=condition=available deploy --all --timeout=600s

# 查看 Pod
kubectl -n argocd get pods
```

### 3.2 访问 UI

```bash
# 端口转发
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 或改 Service 为 NodePort / LoadBalancer
kubectl -n argocd patch svc argocd-server -p '{"spec":{"type":"NodePort"}}'
```

初始密码：

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

用户名 `admin`，登录后**第一件事改密码**。

### 3.3 安装 CLI

```bash
# Linux
curl -sSL -o /usr/local/bin/argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd

# 登录
argocd login localhost:8080
```

### 3.4 生产建议

| 项 | 建议 |
|:--|:--|
| **高可用** | 用 `manifests/ha/install.yaml`，多副本 |
| **TLS** | 前置 Ingress + cert-manager 证书 |
| **SSO** | 接 OIDC（企业微信/Keycloak/GitLab） |
| **存储** | 换掉默认的 internal redis，用外部 Redis |
| **备份** | 定期备份 `argocd` 命名空间的 Secret/ConfigMap |

---

## 四、核心概念

| 概念 | 说明 |
|:--|:--|
| **Application** | 一个应用 = 「哪个 Git 路径」部署到「哪个集群/命名空间」 |
| **Project** | 应用的逻辑分组 + 权限边界（限制能访问哪些仓库和集群） |
| **Source** | 期望状态的来源（Git 仓库 + 路径 + 分支/Tag） |
| **Destination** | 目标集群 + 命名空间 |
| **Sync** | 把 Git 里的期望状态应用到集群 |
| **Healthy** | 资源健康状况（Pod 是否 Ready 等） |
| **Synced** | 集群状态是否与 Git 一致 |
| **Drift** | 偏差：实际状态 ≠ Git 声明 |
| **ApplicationSet** | 批量生成 Application（多环境/多集群） |

**两个独立的状态指标**（新手最容易混）：

```
Sync Status：    Synced / OutOfSync   ← 和 Git 比
Health Status：  Healthy / Degraded   ← 看资源本身
```

可能出现 `Synced` 但 `Degraded`（配置和 Git 一致，但 Pod 起不来），也可能 `OutOfSync` 但 `Healthy`（手改过集群但服务正常）。

---

## 五、创建第一个 Application

### 5.1 用 YAML 声明（推荐，可进 Git）

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/example/deploy-repo.git
    targetRevision: main
    path: apps/my-app

  destination:
    server: https://kubernetes.default.svc
    namespace: my-app

  syncPolicy:
    automated:
      prune: true        # Git 里删掉的资源，集群里也删
      selfHeal: true     # 集群被手改，自动拉回来
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f my-app.yaml
```

### 5.2 用 CLI

```bash
argocd app create my-app \
  --repo https://github.com/example/deploy-repo.git \
  --path apps/my-app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace my-app \
  --sync-policy automated --auto-prune --self-heal
```

### 5.3 三种源类型

| 类型 | 配置 |
|:--|:--|
| **Raw YAML** | `path:` 指向含 `.yaml` 的目录 |
| **Helm** | `source.helm` 指定 chart 与 values |
| **Kustomize** | `path:` 指向含 `kustomization.yaml` 的目录 |

```yaml
# Helm 示例
source:
  repoURL: https://charts.example.com
  chart: my-chart
  targetRevision: 1.2.3
  helm:
    releaseName: my-app
    valueFiles:
      - values-prod.yaml
```

---

## 六、同步策略

### 6.1 手动同步 vs 自动同步

```yaml
syncPolicy:
  automated:
    prune: true         # 删除 Git 中已移除的资源
    selfHeal: true      # 集群被手改后自动纠正
  syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - ApplyOutOfSyncOnly=true     # 只同步有差异的资源，快
```

| 选项 | 作用 |
|:--|:--|
| `prune` | Git 里删掉的资源，集群同步删除 |
| `selfHeal` | 集群被 `kubectl edit` 改过，自动改回 |
| `CreateNamespace` | 目标命名空间不存在时自动创建 |
| `ApplyOutOfSyncOnly` | 只同步 OutOfSync 的资源，提速 |

### 6.2 同步顺序（Sync Waves）

复杂应用有依赖（先建 CRD，再建 Operator，最后建业务）：

```yaml
metadata:
  annotations:
    argoproj.io/sync-wave: "-1"    # 数字小的先执行
```

| Wave | 典型资源 |
|:--|:--|
| `-1` | CRD、Namespace |
| `0` | 基础配置（ConfigMap / Secret） |
| `1` | 中间件（DB / Redis） |
| `2` | 业务应用 |

`sync-wave` 相同的资源并行执行。

### 6.3 Hook（部署前后动作）

```yaml
metadata:
  annotations:
    argoproj.io/hook: PreSync           # 同步前执行
    argoproj.io/hook-delete-policy: HookSucceeded
```

| Hook | 时机 |
|:--|:--|
| `PreSync` | 同步前（如数据库迁移 Job） |
| `Sync` | 同步中 |
| `PostSync` | 同步后（如冒烟测试） |
| `SyncFail` | 同步失败时 |

---

## 七、ApplicationSet：多环境/多集群

一个应用要部署到 dev / staging / prod 三个环境、或者多个集群——逐个写 Application 太累。`ApplicationSet` 用**模板 + 生成器**批量创建。

### 7.1 按目录生成（多环境）

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/example/deploy-repo.git
        revision: main
        directories:
          - path: envs/*
  template:
    metadata:
      name: 'my-app-{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/example/deploy-repo.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
```

仓库结构：

```
envs/
├── dev/
│   └── deployment.yaml
├── staging/
└── prod/
```

→ 自动生成 `my-app-dev`、`my-app-staging`、`my-app-prod` 三个 Application。

### 7.2 按集群生成（多集群）

```yaml
generators:
  - clusters:
      selector:
        matchLabels:
          env: production
```

→ 每个打了 `env=production` 标签的集群，都创建一个 Application。

### 7.3 常用生成器

| 生成器 | 用途 |
|:--|:--|
| **List** | 手写列表（少数固定环境） |
| **Git** | 按 Git 目录/文件生成（**最常用**） |
| **Clusters** | 按注册的集群生成 |
| **Matrix / Merge** | 组合多个生成器（如「集群 × 环境」） |
| **Pull Request** | 每个 PR 生成临时环境（**预览环境**） |

---

## 八、镜像自动更新

GitOps 的经典问题：CI 构建了新镜像，怎么让 ArgoCD 知道？

### 8.1 方案 A：CI 改 Git（最标准）

```bash
# CI 最后一步
git clone https://github.com/example/deploy-repo.git
cd deploy-repo
sed -i "s|image: myapp:.*|image: myapp:${CI_COMMIT_SHORT_SHA}|" apps/my-app/deployment.yaml
git commit -m "chore: bump myapp to ${CI_COMMIT_SHORT_SHA}"
git push
```

→ ArgoCD 检测到 Git 变更 → 自动同步。

### 8.2 方案 B：ArgoCD Image Updater

专门组件，自动检测镜像仓库的新 tag 并写回 Git（或直接改集群）。

```yaml
metadata:
  annotations:
    argoproj.io/image-list: myapp=registry.example.com/myapp
    argoproj.io/myapp.update-strategy: semver
    argoproj.io/myapp.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
```

| 更新策略 | 说明 |
|:--|:--|
| `semver` | 按语义化版本选最高 |
| `latest` | 选最新构建 |
| `digest` | 固定 digest |
| `name` | 按 tag 名匹配 |

### 8.3 方案 C：Kustomize + `kustomize edit set image`

```bash
kustomize edit set image myapp=registry.example.com/myapp:${SHA}
git commit -am "bump image"
git push
```

比 `sed` 更稳，不会误伤其他字段。

---

## 九、多集群管理

### 9.1 注册外部集群

```bash
# 用 kubeconfig 的 context 名
argocd cluster add my-prod-cluster

# 或直接指定 kubeconfig
argocd cluster add my-cluster --kubeconfig ~/.kube/prod.yaml
```

原理：ArgoCD 在目标集群创建 ServiceAccount + ClusterRole，然后用自己的 kubeconfig 访问。

### 9.2 单实例多集群架构

```
           ┌──────────────┐
           │   ArgoCD     │  ← 管理面（一个实例）
           │  (管理集群)   │
           └──────┬───────┘
      ┌───────────┼───────────┐
      ▼           ▼           ▼
  ┌────────┐  ┌────────┐  ┌────────┐
  │ 集群 A  │  │ 集群 B  │  │ 集群 C  │
  │(杭州)   │  │(上海)   │  │(家宽)   │
  └────────┘  └────────┘  └────────┘
```

管理面在一个集群，其余集群挂上去。中小规模推荐这个模式；超大规模（几十集群）才考虑分片。

---

## 十、安全与权限

### 10.1 Project：权限边界

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd
spec:
  sourceRepos:
    - https://github.com/example/team-a-*      # 只能拉这些仓库
  destinations:
    - server: https://kubernetes.default.svc
      namespace: 'team-a-*'                     # 只能部署到这些命名空间
  clusterResourceWhitelist:                     # 集群级资源白名单
    - group: ''
      kind: Namespace
  namespaceResourceBlacklist:                   # 禁止的资源
    - group: ''
      kind: ResourceQuota
```

> ⚠️ **必设项目隔离**：不给边界的话，任何 Application 都能部署到任何命名空间，等于没有权限控制。

### 10.2 RBAC

```csv
# argocd-rbac-cm ConfigMap
p, role:developer, applications, get, team-a/*, allow
p, role:developer, applications, sync, team-a/*, allow
p, role:developer, applications, delete, team-a/*, deny
g, team-a-group, role:developer
```

### 10.3 SSO 登录（OIDC）

```yaml
# argocd-cm
url: https://argocd.example.com
oidc.config: |
  name: GitLab
  issuer: https://gitlab.example.com
  clientID: xxx
  clientSecret: $oidc.gitlab.clientSecret
  requestedScopes: ["openid", "profile", "email", "groups"]
```

---

## 十一、常见坑

| 坑 | 现象 | 解决 |
|:--|:--|:--|
| **同步了但 Pod 起不来** | `Synced` + `Degraded` | 看 Pod 事件，不是 ArgoCD 的问题 |
| **`selfHeal` 把手改改回去** | 手动 `kubectl edit` 无效 | 这是**特性**，改 Git 才对 |
| **`prune: true` 误删资源** | Git 路径写错被清空 | 先在 staging 验证，生产慎用或关掉 |
| **Helm values 不生效** | 没指定 `valueFiles` 或 `valuesObject` | 检查 `source.helm.valueFiles` 路径 |
| **CRD 未就绪就建 CR** | sync 报 `no matches for kind` | 用 `sync-wave: -1` 让 CRD 先建 |
| **Secret 明文进 Git** | 敏感信息泄漏 | 用 Sealed Secrets / SOPS / External Secrets |
| **大仓库拉取慢** | 每次 sync 都全量 clone | 配 `--repo-cache-expiration` 或用 repo-server 缓存 |
| **Image Updater 不生效** | 镜像更新了但没同步 | 检查注解、仓库凭证、更新策略 |
| **多集群网络不通** | Application 一直 Progressing | 检查管理集群到目标集群 6443 连通性 |

---

## 十二、和 CI 的配合

**关键认知：ArgoCD 不是 CI，它只管「部署」这后半段。**

```
CI（Jenkins / GitLab CI）                  CD（ArgoCD）
─────────────────────                    ─────────────────
拉代码 → 编译 → 测试 → 构建镜像
         ↓
    推送镜像仓库
         ↓
   更新部署仓库 YAML  ──────────────►  检测 Git 变更
                                          ↓
                                     同步到集群
                                          ↓
                                     持续调谐（纠偏）
```

| 职责 | 谁负责 |
|:--|:--|
| 代码构建、测试、扫镜像 | **CI** |
| 决定「部署什么版本」 | CI 改 Git（或 Image Updater） |
| 「怎么部署、部署到哪」 | **ArgoCD** |
| 部署后持续保证状态一致 | **ArgoCD** |

> **别让 ArgoCD 干 CI 的活**——它不负责构建镜像。

---

## 十三、一句话总结

> **ArgoCD = 集群里盯着 Git 的眼睛**。
> 你只管往 Git 推期望状态，它负责让集群「长成那个样子」——包括自动纠正有人偷偷手改的地方。

---

## 相关文档

- [[knowledge/devops/gitlab/README\|🦊 GitLab CI/CD]] — CI 侧构建与推送
- [[knowledge/devops/jenkins/README\|🧩 Jenkins]] — 另一套 CI 平台
- [[knowledge/devops/kubernetes/README\|☸️ Kubernetes]] — K8s 基础
- `knowledge/devops/practices/03-CI工程化实战.md` — CI 工程化实战
