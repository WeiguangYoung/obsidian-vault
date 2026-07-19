---
{"dg-publish":true,"permalink":"/knowledge/devops-general/DevOps-7-脚本与开发/","tags":["DevOps","脚本","Python","Shell","Go","Groovy","自动化"],"dg-note-properties":{"date":"2026-07-19","tags":["DevOps","脚本","Python","Shell","Go","Groovy","自动化"]}}
---


# 七、脚本与开发

## 7.1 Python

DevOps 最常用的脚本语言。

### 典型场景

| 场景 | 示例 |
|:----|:------|
| **CI/CD 工具** | 自定义构建脚本、发布脚本 |
| **自动化运维** | 批量操作服务器、资源巡检 |
| **API 封装** | 调用 K8s API、云厂商 API、GitLab API |
| **数据处理** | 日志分析、监控数据聚合 |

### K8s API 调用

```python
from kubernetes import client, config

config.load_kube_config()
v1 = client.CoreV1Api()
pods = v1.list_namespaced_pod(namespace='production')
for pod in pods.items:
    print(f"{pod.metadata.name}: {pod.status.phase}")
```

### 常用库

| 库 | 用途 |
|:----|:------|
| `kubernetes` | K8s Client |
| `requests` | HTTP API 调用 |
| `click` / `typer` | CLI 工具框架 |
| `Jinja2` | 模板渲染（生成配置文件） |
| `pandas` | 数据分析、报表生成 |

## 7.2 Shell (Bash)

Shell 的核心场景：快速、一次性、管道操作。

### 典型场景

| 场景 | 示例 |
|:----|:------|
| **构建脚本** | Makefile 中嵌入 Shell |
| **系统巡检** | 检查磁盘、内存、进程状态 |
| **日志处理** | `grep` + `awk` + `sed` 一站式 |
| **CI/CD Step** | GitLab CI / Jenkins 中的 sh 步骤 |

### 常用技巧

```bash
# 批量重启特定命名空间的 Deployment
kubectl get deploy -n prod -o name | xargs -I {} kubectl rollout restart {} -n prod

# 查询最近一小时的错误日志
kubectl logs -l app=myapp --since=1h | grep ERROR | tail -100

# 检查所有节点的磁盘使用率
df -h | awk '$5+0 > 80 {print $1, $5}'
```

## 7.3 Go

云原生时代的核心语言。

### 典型场景

| 场景 | 示例 |
|:----|:------|
| **K8s Operator** | 自定义 Controller 管理应用生命周期 |
| **CLI 工具** | k9s / kubectl 插件 / 自研工具 |
| **高性能中间件** | 网关、代理、数据同步 |

### client-go 示例

```go
import "k8s.io/client-go/kubernetes"

clientset, _ := kubernetes.NewForConfig(config)
pods, _ := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{})
for _, pod := range pods.Items {
    fmt.Println(pod.Name)
}
```

## 7.4 Groovy

Jenkins Pipeline 专用语言。

```groovy
// Jenkins 共享库
def call(Map config) {
    pipeline {
        agent any
        stages {
            stage('Build') {
                steps { sh config.buildCmd }
            }
        }
        post {
            failure { emailext body: '${BUILD_URL}', subject: 'Build Failed', to: config.owner }
        }
    }
}
```

## 7.5 选型参考

| 语言 | 适合 | 不适合 |
|:----|:------|:------|
| **Python** | CI/CD 工具、API 封装、数据处理 | 高性能服务、系统级工具 |
| **Shell** | 构建脚本、系统巡检、一次性操作 | 复杂业务逻辑、跨平台 |
| **Go** | K8s Operator、CLI 工具、中间件 | 快速原型验证 |
| **Groovy** | Jenkins Pipeline | 任何非 Jenkins 的场景 |
