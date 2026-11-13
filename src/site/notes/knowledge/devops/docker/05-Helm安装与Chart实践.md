---
{"dg-publish":true,"permalink":"/knowledge/devops/docker/05-Helm安装与Chart实践/","title":"Helm 使用：从安装到 Chart 编写","tags":["helm","k8s","chart","容器","编排","DevOps"],"dg-note-properties":{"category":"Docker / 容器 / DevOps","title":"Helm 使用：从安装到 Chart 编写","date":"2026-11-10","tags":["helm","k8s","chart","容器","编排","DevOps"]}}
---


# Helm 使用：从安装到 Chart 编写

> Helm 是 K8s 的包管理器，但它的价值不在"少写 YAML"，而在**模板化 + 版本化 + 可回滚**——同一套 Chart 能部署到 dev/staging/prod 三套环境，参数不同而已。
> 一句话主线：**Helm = 模板引擎（Go template）+ 包（Chart）+ 发布记录（Release）。理解这三样，剩下的都是语法。**

---

## 一、安装

### 1.1 脚本安装（推荐）

```bash
curl -fsSL -o /tmp/get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 /tmp/get_helm.sh
/tmp/get_helm.sh

helm version
```

### 1.2 apt 安装

```bash
curl https://baltocdn.com/helm/signing.asc | sudo gpg --dearmor -o /usr/share/keyrings/helm.gpg
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" \
  | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install -y helm
```

> 老教程里的 `apt-key add` 在 Debian 12 / Ubuntu 24 已废弃，用上面的 `signed-by` 写法。

### 1.3 国内网络问题

`raw.githubusercontent.com` 和 `baltocdn.com` 国内都可能卡。**最省事的办法**：直接下二进制。

```bash
# 从 GitHub Releases 下（需要能访问）
wget https://get.helm.sh/helm-v3.16.3-linux-amd64.tar.gz
tar -zxvf helm-v3.16.3-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
helm version
```

---

## 二、核心概念（三句话说清）

| 概念 | 是什么 |
|:--|:--|
| **Chart** | 一个目录，里面是模板 + 默认值 + 元数据。相当于一个"安装包" |
| **Release** | Chart 部署到集群后的一次**实例**。同名 Chart 可以部署多次（release 名不同）|
| **Repository** | Chart 的存放仓库（HTTP 服务 + index.yaml）|

```
Chart（模板）  --helm install-->  Release（集群里的实例）
     ↑                                    ↓
  可版本化                            helm rollback 可回滚
```

---

## 三、仓库管理

### 3.1 添加仓库

```bash
# 官方
helm repo add stable https://charts.helm.sh/stable          # 已归档，仅历史
helm repo add bitnami https://charts.bitnami.com/bitnami    # 最常用

# 国内镜像（bitnami 国内访问慢时用）
helm repo add bitnami https://charts.bitnami.com/bitnami --force-update

# 更新索引
helm repo update

# 列出已加仓库
helm repo list

# 搜索
helm search repo nginx
helm search repo nginx --versions      # 看所有版本
```

### 3.2 常用 Chart

```bash
helm install my-pg bitnami/postgresql
helm install my-redis bitnami/redis
helm install my-nginx bitnami/nginx
helm install prometheus prometheus-community/kube-prometheus-stack
```

---

## 四、安装 / 升级 / 回滚

### 4.1 安装

```bash
# 最简
helm install my-release bitnami/nginx

# 指定 namespace（不存在就创建）
helm install my-release bitnami/nginx -n myapp --create-namespace

# 自定义参数
helm install my-release bitnami/nginx \
  --set service.type=NodePort \
  --set replicaCount=3

# 用 values 文件（推荐，参数多了就别 --set）
helm install my-release bitnami/nginx -f my-values.yaml

# 渲染但不安装（先看看生成什么）
helm install my-release bitnami/nginx --dry-run --debug
```

> **`--dry-run --debug` 是 Helm 最重要的命令**。参数写错了直接装，可能把集群搞乱；先 dry-run 看渲染结果。

### 4.2 查看

```bash
helm list                          # 当前 namespace
helm list -A                       # 所有 namespace
helm status my-release -n myapp    # 状态 + NOTES
helm get values my-release -n myapp          # 你传的 values
helm get values my-release -n myapp --all    # 含默认值（合并后）
helm get manifest my-release -n myapp        # 渲染出的 K8s YAML
helm history my-release -n myapp             # 版本历史
```

### 4.3 升级

```bash
helm upgrade my-release bitnami/nginx -f new-values.yaml
helm upgrade my-release bitnami/nginx --set replicaCount=5

# 加 --install：不存在就装，存在就升（CI 里常用）
helm upgrade --install my-release bitnami/nginx -f values.yaml -n myapp
```

### 4.4 回滚

```bash
helm rollback my-release 1 -n myapp      # 回到 revision 1
helm history my-release -n myapp         # 先看有哪些 revision
```

### 4.5 卸载

```bash
helm uninstall my-release -n myapp
helm uninstall my-release -n myapp --keep-history    # 保留历史，可回滚
```

---

## 五、values 优先级（必须搞清楚）

从低到高：

```
1. Chart 里的 values.yaml           （默认值）
2. -f 指定的文件（多个 -f 后者覆盖前者）
3. --set / --set-string / --set-file
4. --set 里后写的覆盖先写的
```

**`-f` 和 `--set` 混用的坑**：

```bash
# 这样 replicaCount=3 会覆盖 values.yaml 里的值
helm install x ./mychart -f values.yaml --set replicaCount=3
```

**多层 -f 叠加**：

```bash
helm install x ./mychart \
  -f values.yaml \           # 基础
  -f values-prod.yaml \      # 生产覆盖
  -f values-secret.yaml      # 密钥覆盖（不进仓库）
```

### --set 的类型陷阱

```bash
--set replicas=3            # 数字
--set name=123              # ❗ 变成字符串 "123"，但 YAML 里需要数字时会报错
--set-string name=123       # 强制字符串
--set image.tag=1.2.3       # 会被解析成 "1.2.3"（字符串，OK）
--set image.tag=1.20       # ❗ 可能被解析成 1.2（浮点）→ 用 --set-string
```

> **凡是版本号、tag、字符串数字，一律用 `--set-string`。** 这是 Helm 最常见的翻车点。

**数组 / 嵌套**：

```bash
--set 'ingress.hosts[0]=a.com'
--set 'ingress.hosts[1]=b.com'
--set 'env[0].name=FOO' --set 'env[0].value=bar'
```

---

## 六、写一个 Chart

### 6.1 生成骨架

```bash
helm create mychart
```

```
mychart/
├── Chart.yaml           # 元数据（名字、版本、依赖）
├── values.yaml          # 默认值
├── charts/              # 子 chart（依赖）
├── templates/
│   ├── _helpers.tpl     # 模板函数（不渲染成文件）
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   ├── NOTES.txt        # 安装后提示
│   └── tests/
│       └── test-connection.yaml
└── .helmignore
```

### 6.2 Chart.yaml

```yaml
apiVersion: v2
name: mychart
description: My application chart
type: application        # 或 library
version: 0.1.0           # Chart 版本（改模板必须升）
appVersion: "1.2.3"      # 应用版本（改业务代码升这个）
```

> **`version` 和 `appVersion` 的区别**：前者是 Chart 本身版本（模板改动），后者是部署的应用版本（镜像 tag）。**两个都要维护，别混用。**

### 6.3 values.yaml 设计原则

```yaml
# 顶层按组件分组，不要平铺
replicaCount: 2

image:
  repository: myapp/api
  tag: ""                # 留空，由 --set-string 覆盖
  pullPolicy: IfNotPresent

resources:
  limits: { cpus: "1", memory: 512Mi }
  requests: { cpus: "100m", memory: 128Mi }

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false          # 关键：所有可选组件用 enabled 开关
  className: nginx
  hosts:
    - host: chart-example.local
      paths: [{ path: /, pathType: Prefix }]

env: []
```

**三条设计原则**：

| 原则 | 说明 |
|:--|:--|
| 可选组件用 `enabled` | ingress/hpa/pdb 全部带开关，默认关 |
| 不给密钥设默认值 | 密码类字段留空，由 `-f values-secret.yaml` 提供 |
| 提供完整注释 | `values.yaml` 就是用户文档，每行都要 `#` 说明 |

### 6.4 模板常用语法

**用 values**：

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

**条件**：

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}
```

**循环**：

```yaml
{{- range .Values.env }}
- name: {{ .name }}
  value: {{ .value | quote }}
{{- end }}
```

**默认值**：

```yaml
{{ .Values.image.tag | default .Chart.AppVersion }}
{{ .Values.port | default 8080 }}
```

### 6.5 _helpers.tpl（命名规范）

```yaml
{{/*
标准名称
*/}}
{{- define "mychart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
完整名称：release 名 + chart 名，避免多 release 冲突
*/}}
{{- define "mychart.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
通用 labels
*/}}
{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.name" . }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
selector labels（不能含 version，否则升级会冲突）
*/}}
{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

> ⚠️ **最常见的设计错误**：`selectorLabels` 里放了 `helm.sh/chart` 或 version。K8s 的 `selector` 是不可变字段，**只要 version 变了 upgrade 就会失败**（`field is immutable`）。

### 6.6 NOTES.txt

安装完打印的提示：

```
应用 {{ .Release.Name }} 已部署。

访问方式：
{{- if .Values.ingress.enabled }}
  http://{{ (index .Values.ingress.hosts 0).host }}
{{- else }}
  kubectl port-forward svc/{{ include "mychart.fullname" . }} 8080:{{ .Values.service.port }}
{{- end }}
```

---

## 七、调试 Chart

```bash
# 渲染模板（不装）
helm template mychart ./mychart

# 带 values 渲染
helm template mychart ./mychart -f values.yaml

# 只渲染某个模板
helm template mychart ./mychart -s templates/deployment.yaml

# 语法检查
helm lint ./mychart

# 严格模式（缺值会报错）
helm lint ./mychart --strict

# 装到集群但失败会回滚验证
helm install --dry-run --debug mychart ./mychart
```

**推荐工作流**：

```
改模板 → helm template 看渲染 → helm lint → helm install --dry-run --debug → 真装
```

> `helm template` 是最快的反馈循环（不连集群），`--debug` 能看渲染失败的具体行号。

---

## 八、Chart 依赖

### 8.1 Chart.yaml 里声明

```yaml
dependencies:
  - name: postgresql
    version: "15.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled         # 可选：由 values 控制是否启用
  - name: redis
    version: "19.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

### 8.2 拉依赖

```bash
helm dependency update ./mychart     # 下载到 charts/
helm dependency build ./mychart      # 按 Chart.lock 精确还原
helm dependency list ./mychart
```

> `Chart.lock` 必须提交到 git——它锁定依赖的精确版本，保证不同机器构建结果一致。

### 8.3 依赖的 values

子 chart 的 values 在**顶层用子 chart 名作 key**：

```yaml
# 父 chart 的 values.yaml
postgresql:
  enabled: true
  auth:
    database: myapp
    password: ""        # 留空，由 secrets 文件提供
  primary:
    persistence:
      size: 20Gi
```

---

## 九、常见报错

| 报错 | 原因 | 解法 |
|:--|:--|:--|
| `Error: INSTALLATION FAILED: cannot re-use a name that is still in use` | release 名已存在 | 换名，或 `helm uninstall` 后重装 |
| `field is immutable` | 改了 selector labels | `helm uninstall` 后重装（StatefulSet 尤其常见）|
| `Error: UPGRADE FAILED: another operation is in progress` | 上次操作卡住 | `helm rollback` 或删 secret `sh.helm.release.v1.<name>.v<N>` |
| `nil pointer evaluating interface {}.xxx` | values 里没这个字段 | values.yaml 补上，或用 `default` |
| `error converting YAML to JSON` | 模板渲染出非法 YAML | `helm template --debug` 看具体行 |
| `no repositories found` | 没 `helm repo add` | 先加仓库 |
| image.tag 变成 1.2 而不是 1.20 | `--set` 把数字解析成 float | 用 `--set-string` |
| `rendered manifests contain a resource that already exists` | 资源已存在且不归这个 release 管 | 加 `--force` 或先删资源 |

### 卡住的 release 怎么救

```bash
# 看状态
helm list -A | grep pending

# 方案 1：回滚
helm rollback <release> <上一个版本> -n <ns>

# 方案 2：强制标记为已失败并重试
# 先找 secret
kubectl get secret -n <ns> | grep sh.helm.release

# 方案 3：直接 uninstall 重装
helm uninstall <release> -n <ns>
```

---

## 十、在 CI 里用 Helm

```bash
# 1. 校验
helm lint ./chart
helm template app ./chart -f values-prod.yaml > /tmp/rendered.yaml

# 2. 用 kubeconform / kubeval 校验渲染结果
kubeconform -strict -summary /tmp/rendered.yaml

# 3. 差异化预览（v3 需要插件）
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade app ./chart -f values-prod.yaml

# 4. 部署（幂等）
helm upgrade --install app ./chart \
  -f values-prod.yaml \
  -n prod --create-namespace \
  --atomic \                # 失败自动回滚
  --timeout 5m \
  --wait                    # 等资源 ready

# 5. 验证
helm status app -n prod
kubectl rollout status deploy/app -n prod
```

**`--atomic --wait` 是关键组合**：部署失败自动回滚到上一个版本，不用人工救火。

---

## 十一、一句话总结

> **Helm 的价值不是少写几个 YAML，而是「同一份模板，多套环境，可版本化、可回滚」。**
> 三个必记住的点：**`--set-string` 处理版本号**、**selectorLabels 不能含 version**、**`upgrade --install --atomic --wait` 是生产标配**。
