---
{"dg-publish":true,"permalink":"/knowledge/tools/Airflow部署方案/","tags":["DevOps","Airflow","K8s","定时任务","调度系统","汽车嵌入式","华为云"],"dg-note-properties":{"date":"2026-07-21","updated":"2026-08-05","tags":["DevOps","Airflow","K8s","定时任务","调度系统","汽车嵌入式","华为云"]}}
---


# Airflow 部署方案

> 从 [[notes/1-工作/项目经历/04-CCE集群管理\|04-CCE集群管理]] 第十二章独立出来。集群内大量每日定时任务（安全扫描报告聚合、Fuzz 报告汇总、MySQL 备份、ETL 同步等）统一由 **Airflow on K8s** 调度管理。
>
> 合并说明：2026-08-05 将原 [[Airflow-K8s-单节点验证方案\|Airflow-K8s-单节点验证方案]] 内容并入第十四章（验证方案），保留单节点验证流程的完整记录。

## 一、选型

选用 **Airflow on K8s（KubernetesExecutor）**，每个 Task 跑独立 Pod，利用 K8s 弹性。

核心优势：DAG 依赖可视化、backfill 批量补数、on_failure_callback 原生告警、Web UI 一键重跑。

### 1.1 各 Executor 对比

Airflow 的 **Executor（执行器）** 决定 Task 实际在哪里、以什么方式跑。常见几种：

| Executor | 运行方式 | 弹性 | 资源隔离 | 适用场景 |
|:---------|:---------|:----:|:--------:|:---------|
| **SequentialExecutor** | 单进程串行 | ❌ | ❌ | 仅本地调试，生产禁用 |
| **LocalExecutor** | 单机多进程并行 | ❌ | 一般 | 单机小规模，无分布式需求 |
| **CeleryExecutor** | Worker 常驻，任务下发到队列（Redis/RabbitMQ）| ⚠️ 需手动扩缩 Worker | 一般（共享 Worker）| 大规模稳定负载、长驻任务池 |
| **KubernetesExecutor** | 每个 Task 起独立 Pod | ✅ 真弹性 | ✅ Pod 级隔离 | K8s 环境、任务稀疏、需强隔离 |
| **LocalKubernetesExecutor** | 混合：本地跑短任务 + Pod 跑大任务 | ✅ 部分 | 混合 | 兼顾轻量任务与弹性 |
| **CeleryKubernetesExecutor** | 混合：Celery 跑常规 + Pod 跑重任务 | ✅ 部分 | 混合 | 已有 Celery 集群又想借 K8s 弹性 |

**本方案选型理由**：任务多为**每日稀疏定时**（安全扫描报告聚合、Fuzz 汇总、MySQL 备份等），不是持续高并发负载。
- KubernetesExecutor 任务用完即销毁 Pod，**空闲零成本**，比 Celery 常驻 Worker 更省资源；
- K8s 环境下天然复用集群调度与隔离能力，无需额外维护 Redis/RabbitMQ + Worker 池。

> 若后续出现**高频 / 长时 / 密集**任务（如秒级触发、批量 ETL 洪峰），可切 **CeleryExecutor** 或 **CeleryKubernetesExecutor**（见 §3.5）。

## 二、节点 & 资源

| 节点 | 角色 | CPU | 内存 | 硬盘 |
|:-----|:-----|:---:|:----:|:----:|
| airflow-1 | Webserver + Scheduler（HA） | 6u | 12G | 100G |
| airflow-2 | Worker + Triggerer + PostgreSQL | 6u | 12G | 100G |

> 来源：监控组件压缩配额让出 12u/24G/300G，Airflow 占用 12u/24G/200G，剩余 100G 留给 Grafana 备份盘。
> 详见 [[DevOps资源量预估\|DevOps资源量预估]]

### 资源预估明细

| 组件 | CPU | 内存 |
|:-----|:---:|:----:|
| Webserver | 500m | 1Gi |
| Scheduler | 1 | 2Gi |
| Triggerer | 200m | 512Mi |
| PostgreSQL | 500m | 1Gi |
| **每个 Task Pod** | **按需** | **按需** |

> 生产建议**外接 RDS PostgreSQL**，不要用内置的。

## 三、部署（Helm Chart）

### 3.1 一键部署

```bash
helm repo add apache-airflow https://airflow.apache.org
helm repo update

kubectl create namespace airflow

helm install airflow apache-airflow/airflow \
  --namespace airflow \
  --set executor=KubernetesExecutor \
  --set workers.persistence.enabled=true \
  --set dags.persistence.enabled=true \
  --set dags.gitSync.enabled=true \
  --set dags.gitSync.repo=https://github.com/your-org/airflow-dags.git \
  --set dags.gitSync.branch=main \
  --set webserver.service.type=LoadBalancer \
  --set webserver.nodeSelector.workload=airflow \
  --set scheduler.nodeSelector.workload=airflow \
  --set workers.nodeSelector.workload=airflow \
  --set triggerer.nodeSelector.workload=airflow \
  --set postgresql.nodeSelector.workload=airflow
```

### 3.2 部署说明

| 配置 | 说明 |
|:-----|:-----|
| `executor=KubernetesExecutor` | 每个 Task 跑独立 Pod，真正利用 K8s 弹性 |
| `dags.gitSync.enabled=true` | DAG 代码从 Git 仓库拉取，无 PVC |
| `dags.persistence.enabled=true` | DAG 解析缓存保留 |
| `workers.persistence.enabled=true` | Worker 日志持久化 |
| `nodeSelector.workload=airflow` | 绑定 airflow-1/airflow-2 节点 |

### 3.3 高可用

- **Webserver**：跑 2 副本，通过 ELB Service 负载均衡
- **Scheduler**：跑 2 副本，Active-Standby（K8s 选举）
- **Worker**：按需扩缩，HPA 监听 Task 队列积压
- **Triggerer**：跑 2 副本，异步 Trigger 多活

> 配置示例：`--set webserver.replicas=2 --set scheduler.replicas=2`

### 3.4 Airflow CeleryExecutor 部署（备选）

当任务量增长到 KubernetesExecutor 撑不住（Pod 创建延迟累积、Scheduler 被 K8s API 拖垮）时，改用 CeleryExecutor：**Scheduler 只负责发任务，常驻 Worker 从队列取活执行**。

**额外依赖**：Redis（Broker）+ 常驻 Celery Worker + 可选 Flower（监控）。

```bash
helm upgrade --install airflow apache-airflow/airflow \
  -n mon-af --create-namespace \
  --set executor=CeleryExecutor \
  --set redis.enabled=true \
  --set workers.replicas=3 \
  --set workers.nodeSelector.workload=airflow \
  --set workers.resources.requests.cpu=1 \
  --set workers.resources.requests.memory=2Gi \
  --set flower.enabled=true
```

**资源形态变化**：

| 组件 | KubernetesExecutor | CeleryExecutor |
|:-----|:-------------------|:---------------|
| Worker | 无（Task 即 Pod）| **常驻 3 副本**（HPA 可扩）|
| Broker | 无 | Redis / RabbitMQ |
| 空闲成本 | ~0 | Worker 常驻占用 |
| 任务启动延迟 | 5~10s（起 Pod）| 低（Worker 已就绪）|

**扩缩容**：Celery 侧靠 `workers.replicas` 或 KEDA/HPA 按队列长度自动扩缩，不必为每个任务起 Pod。

**混合方案（CeleryKubernetesExecutor）**：常规任务走 Celery 常驻 Worker，重型任务（大内存 / 需隔离）走独立 Pod，两全其美：

```bash
--set executor=CeleryKubernetesExecutor
```

### 3.5 节点标签 & 污点

```bash
kubectl label node airflow-{1..2} workload=airflow
kubectl taint node airflow-{1..2} workload=airflow:NoSchedule
```

> Airflow 节点独享，不允许其他业务 Pod 调度上来。

## 四、命名空间 & 网络

### 4.1 命名空间

| 命名空间      | 内容                                                       |
| :-------- | :------------------------------------------------------- |
| `airflow` | Airflow（Webserver + Scheduler + Worker + Triggerer + PG） |

### 4.2 访问规则

| 规则 | 说明 |
|:-----|:-----|
| Airflow → monitoring:9090 | 推送 Task 指标（`airflow_stats_exporter`） |
| Airflow → 裸机 MySQL:3306 | 写入调度结果（ExternalName Service） |
| Airflow → 裸机 Jenkins:8080 | 触发 Pipeline（DNS: `jenkins.automotive.internal`） |
| Airflow → SWR 内网 | 拉取 Task 镜像 |

### 4.3 NetworkPolicy 原则

- 默认跨 ns 不通，NetworkPolicy 白名单放行
- `airflow` ns → `monitoring` ns：只开 9090 出口
- `airflow` ns → 裸机：白名单 3306、8080

## 五、存储

| 用途 | 华为云 | 说明 |
|:-----|:-------|:-----|
| Airflow DAGs | gitSync（无 PVC） | DAG 代码仓库拉取 |
| Airflow 元数据 | EVS SSD | PostgreSQL 数据持久化 |
| Task 日志 | SFS Turbo（RWX） | 跨 Worker 共享 |
| DAG 备份 | EVS SSD | 50G，存储导入导出 JSON |

## 六、DAG 示例（每日安全扫描报告聚合）

```python
from airflow import DAG
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import KubernetesPodOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-team',
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'email_on_failure': True,
}

with DAG(
    dag_id='daily_security_report',
    default_args=default_args,
    schedule='0 3 * * *',  # 每日 3:00（扫描任务完成后）
    start_date=datetime(2026, 7, 1),
    catchup=False,
    tags=['security', 'daily'],
) as dag:

    collect = KubernetesPodOperator(
        task_id='collect_scan_results',
        name='collect-scan',
        namespace='airflow',
        image='my-registry/scan-collector:latest',
        cmds=['python', '/app/collect.py'],
        get_logs=True,
    )

    aggregate = KubernetesPodOperator(
        task_id='aggregate_report',
        name='aggregate',
        namespace='airflow',
        image='my-registry/report-aggregator:latest',
        cmds=['python', '/app/aggregate.py'],
        get_logs=True,
    )

    notify = KubernetesPodOperator(
        task_id='notify_dingtalk',
        name='notify',
        namespace='airflow',
        image='my-registry/dingtalk-notifier:latest',
        cmds=['python', '/app/notify.py'],
        get_logs=True,
    )

    collect >> aggregate >> notify
```

## 七、手动重跑

| 场景 | 操作 |
|:-----|:-----|
| 重跑单个 DAG 实例 | Web UI → DAG → 选择 execution_date → 点击 ▶ |
| 重跑某个 Task | Web UI → DAG → 点击 Task → "Clear" |
| 批量补数（按日期范围） | CLI: `airflow dags backfill daily_security_report --start-date 2026-07-14 --end-date 2026-07-21` |
| CLI 触发 | `airflow dags trigger daily_security_report` |
| 清空失败 Task | `airflow tasks clear daily_security_report --start-date 2026-07-21 --end-date 2026-07-21` |

## 八、告警

### 8.1 钉钉/企微 Webhook（Task 级别）

```python
def task_failure_alert(context):
    import requests
    webhook = "https://oapi.dingtalk.com/robot/send?access_token=***"
    requests.post(webhook, json={
        "msgtype": "markdown",
        "markdown": {
            "title": f"❌ Task 失败: {context['task_instance'].task_id}",
            "text": (
                f"**DAG**: {context['dag'].dag_id}\n"
                f"**Task**: {context['task_instance'].task_id}\n"
                f"**执行时间**: {context['ts']}\n"
                f"**日志**: {context['task_instance'].log_url}"
            )
        }
    })

default_args = {
    'on_failure_callback': task_failure_alert,
    'on_retry_callback': task_failure_alert,
}
```

### 8.2 Prometheus + AlertManager（集群级别）

| 规则 | 条件 | 级别 |
|:-----|:-----|:-----|
| AirflowDAGFailed | dag_run_status="failed"，5min | warning |
| AirflowSchedulerDown | up == 0，3min | critical |
| AirflowTaskQueueBacklog | task_queue_length > 100，10min | warning |

### 8.3 告警分级

| 级别 | 场景 | 响应 |
|:-----|:-----|:-----|
| P0 | Scheduler/Webserver 全挂 | 5 分钟内 |
| P1 | 单个 DAG 连续 3 次失败 | 15 分钟内 |
| P2 | 单次 Task 失败 | 1 小时内 |
| P3 | Task 耗时超阈值 | 工作时间处理 |

## 九、监控

### 9.1 关键指标

| 指标 | PromQL 思路 |
|:-----|:-------------|
| DAG 成功率 | `sum(dag_run_success) / sum(dag_run_total)` |
| Task 平均耗时 | `avg(task_duration_seconds)` |
| Worker Pod 数量 | `kube_deployment_status_replicas{deployment="airflow-worker"}` |
| 队列积压 | `airflow_task_queue_length` |
| Scheduler 健康 | `up{job="airflow-scheduler"}` |

### 9.2 Grafana Dashboard

| Dashboard | 内容 |
|:----------|:-----|
| Airflow 总览 | DAG 成功率、Task 耗时、Worker Pod、队列积压 |
| DAG 详情 | 各 DAG 执行历史、Gantt 图、失败原因 |
| Worker 资源 | Worker Pod CPU/内存、K8s 事件 |
| 数据库 | PostgreSQL 连接数、QPS、慢查询 |

## 十、任务迁移清单

| 原任务 | 原方式 | 迁移后 |
|:-------|:-------|:-------|
| MySQL 每日全量备份 | K8s CronJob | Airflow DAG `daily_mysql_backup` |
| 安全扫描报告聚合 | Jenkins 定时任务 | Airflow DAG `daily_security_report` |
| Fuzz 报告汇总 | K8s CronJob | Airflow DAG `daily_fuzz_summary` |
| 监控数据归档 | K8s CronJob | Airflow DAG `daily_metrics_archive` |
| Jenkins 构建结果通知 | Post-build step | Airflow DAG 触发 + 回调 |

> **迁移原则**：先迁**有依赖**的（DAG 价值高），后迁**孤立**的（K8s CronJob 即可）。

## 十一、Airflow 优劣势

### ✅ 优势

- 真正弹性（任务用完即销毁 Pod）
- 资源隔离好（每个 Task 独立 Pod）
- DAG 可视化 + 补数（`backfill`）
- 插件丰富（Operator 多：K8sPodOperator、MySqlOperator、DingdingOperator 等）
- 完整运行历史，审计/合规友好

### ⚠️ 劣势

- 部署复杂（Helm Chart 很多参数）
- K8s API 调用频繁，Scheduler 压力大
- 学习成本高（DAG + Python）
- 启动 Pod 有延迟（5~10 秒），不适合秒级调度

## 十二、备份

| 备份对象 | 策略 | 保留 |
|:---------|:-----|:-----|
| PostgreSQL（元数据） | 每日 03:00 pg_dump → OBS | 7 天 |
| DAG 仓库 | GitLab 托管，多副本 | 永久 |
| Airflow 配置 | Helm values.yaml 入 Git | 永久 |
| Task 日志 | SFS Turbo → OBS 冷备 | 30 天 |

## 十三、参考

- 官方文档：https://airflow.apache.org/docs/helm-chart/
- 部署方案总览：[[notes/1-工作/项目经历/04-CCE集群管理\|04-CCE集群管理]]
- 资源量预估：[[DevOps资源量预估\|DevOps资源量预估]]

## 十四、单节点验证方案

> 在单节点 K8s 上模拟完整调度流程：Airflow DAG 按需拉起伪代码 Pod（安全扫描 / 工具服务 / Fuzz），SmartIDE 部署 WebIDE 开发环境，Prometheus + Grafana 采集指标。
> Jenkins / MySQL 不在本次验证范围内。
> 合并自原 [[Airflow-K8s-单节点验证方案\|Airflow-K8s-单节点验证方案]]（2026-08-05）。

### 14.1 验证目标

| 目标 | 验证内容 |
|:-----|:--------|
| ① Airflow 部署 | KubernetesExecutor 在单节点 K8s 上正常运行 |
| ② DAG 触发 Pod | 每个 Task 自动创建独立 Pod，跑完销毁 |
| ③ 分组调度 | Pod 通过 nodeSelector 落到指定标签节点 |
| ④ 伪代码业务 | 安全扫描(7项)、工具服务(4项)、Fuzz(2项) 全部覆盖 |
| ⑤ SmartIDE | WebIDE 开发环境部署，浏览器访问 VS Code |
| ⑥ 监控采集 | Prometheus 采集 Airflow metrics + 业务 Pod metrics |
| ⑦ AlertManager 告警 | 模拟 DAG 失败 → 告警触发 |

### 14.2 环境准备

#### 14.2.1 硬件要求

| 资源 | 最低 | 建议 |
|:-----|:---:|:---:|
| CPU | 4u | 8u |
| 内存 | 16G | 24G |
| 硬盘 | 50G | 100G |

#### 14.2.2 单节点 K8s

任选一种：

```bash
# 方案 A：k3s（推荐，轻量）
curl -sfL https://get.k3s.io | sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# 方案 B：minikube
minikube start --cpus=4 --memory=16g --disk-size=50g

# 方案 C：kind
kind create cluster --config kind-config.yaml
```

#### 14.2.3 节点标签（模拟分组）

```bash
# 单节点打上所有业务标签（模拟分组调度）
kubectl label node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') \
  workload=security-scan \
  workload=tools \
  workload=fuzz-test \
  workload=mon-af
```

#### 14.2.4 命名空间

```bash
kubectl create namespace security-scan
kubectl create namespace tools
kubectl create namespace fuzz-test
kubectl create namespace mon-af
```

### 14.3 Airflow 部署（验证版）

#### 14.3.1 Helm 安装（单节点精简版）

```bash
helm repo add apache-airflow https://airflow.apache.org
helm repo update

# 单节点精简版（去掉 HA，降低资源消耗）
helm install airflow apache-airflow/airflow \
  --namespace mon-af \
  --set executor=KubernetesExecutor \
  --set createUserJob.useHelmHooks=false \
  --set webserver.replicas=1 \
  --set scheduler.replicas=1 \
  --set workers.replicas=1 \
  --set triggerer.replicas=1 \
  --set dagProcessor.replicas=1 \
  --set statsd.enabled=false \
  --set flower.enabled=false \
  --set postgresql.persistence.size=2Gi \
  --set logs.persistence.enabled=false \
  --set dags.gitSync.enabled=false \
  --set defaultAirf…tory=apache/airflow \
  --set defaultAirflowTag=2.8.0 \
  --set webserver.defaultUser.username=admin \
  --set webserver.defaultUser.password=admin \
  --set webserver.defaultUser.email=admin@test.com \
  --set webserver.defaultUser.firstName=Admin \
  --set webserver.defaultUser.lastName=User \
  --set webserver.service.type=NodePort
```

#### 14.3.2 等待就绪

```bash
kubectl wait --for=condition=ready pod \
  -l component=webserver -n mon-af --timeout=300s

# 获取 Web UI 地址
echo "http://$(kubectl get node -o jsonpath='{.items[0].status.addresses[0].address}'):$(kubectl get svc airflow-webserver -n mon-af -o jsonpath='{.spec.ports[0].nodePort}')"
```

#### 14.3.3 RBAC 授权（允许 Airflow 在业务 ns 创建 Pod）

```bash
# 为每个业务 ns 创建 Role + RoleBinding
for ns in security-scan tools fuzz-test; do
  kubectl -n $ns create role airflow-pod-manager \
    --verb=get,list,create,delete,watch \
    --resource=pods,pods/log

  kubectl -n $ns create rolebinding airflow-manage-pods \
    --role=airflow-pod-manager \
    --serviceaccount=mon-af:airflow-worker
done
```

### 14.4 伪代码业务镜像

#### 14.4.1 Dockerfile（通用伪代码模板）

```dockerfile
FROM alpine:3.19
COPY run.sh /run.sh
RUN chmod +x /run.sh
ENTRYPOINT ["/run.sh"]
```

#### 14.4.2 run.sh（接收参数，打印日志）

```bash
#!/bin/sh
TASK_NAME=${TASK_NAME:-"unknown"}
SLEEP_SEC=${SLEEP_SEC:-10}

echo "========================================="
echo "Task:   $TASK_NAME"
echo "Node:   $(hostname)"
echo "Start:  $(date '+%Y-%m-%d %H:%M:%S')"
echo "========================================="

# 模拟工作
for i in $(seq 1 $SLEEP_SEC); do
  echo "[$TASK_NAME] step $i/$SLEEP_SEC ..."
  sleep 1
done

echo "========================================="
echo "Result: SUCCESS"
echo "End:    $(date '+%Y-%m-%d %H:%M:%S')"
echo "========================================="
```

#### 14.4.3 构建 & 导入镜像

```bash
# 构建
docker build -t mock-task:latest .

# k3s 导入（免推 registry）
docker save mock-task:latest | sudo k3s ctr images import -

# 或 minikube
eval $(minikube docker-env)
docker build -t mock-task:latest .
```

### 14.5 Airflow DAG（验证版）

#### 14.5.1 DAG 代码（dag_mock_workflow.py）

```python
from airflow import DAG
from airflow.providers.cncf.kubernetes.operators.pod import KubernetesPodOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=2),
}

# 通用任务工厂
def mock_task(name, namespace, cpu, mem, sleep_sec):
    return KubernetesPodOperator(
        task_id=name,
        namespace=namespace,
        image='mock-task:latest',
        env_vars={
            'TASK_NAME': name,
            'SLEEP_SEC': str(sleep_sec),
        },
        node_selector={'workload': namespace},  # 模拟分组调度
        resources={
            'requests': {'cpu': cpu, 'memory': f'{mem}Mi'},
            'limits':   {'cpu': cpu, 'memory': f'{mem}Mi'},
        },
        is_delete_operator_pod=True,  # 跑完销毁
        startup_timeout_seconds=120,
    )

with DAG(
    dag_id='daily_mock_workflow',
    default_args=default_args,
    schedule='@daily',
    start_date=datetime(2026, 7, 21),
    catchup=False,
    tags=['mock', 'validation'],
) as dag:

    # ── 安全扫描 7 项（错峰调度） ──
    secret_scan     = mock_task('secret_scan',     'security-scan', '100m', 128, 8)
    sudo_check      = mock_task('sudo_check',      'security-scan', '100m', 128, 6)
    crypto_check    = mock_task('crypto_check',    'security-scan', '100m', 128, 6)
    data_check      = mock_task('data_check',      'security-scan', '100m', 128, 5)
    port_scan       = mock_task('port_scan',       'security-scan', '100m', 128, 5)
    binary_scan     = mock_task('binary_scan',     'security-scan', '100m', 128, 5)
    virus_scan      = mock_task('virus_scan',      'security-scan', '200m', 256, 10)

    # ── 工具服务 4 项 ──
    auto_tag        = mock_task('auto_tag',        'tools',          '50m',  64,  4)
    mail_notify     = mock_task('mail_notify',     'tools',          '50m',  64,  4)
    xsan_sync       = mock_task('xsan_sync',       'tools',          '50m',  64,  5)
    nmap_scan       = mock_task('nmap_scan',       'tools',          '50m',  64,  5)

    # ── Fuzz 2 项 ──
    dtfuzz          = mock_task('dtfuzz',          'fuzz-test',      '200m', 256, 12)
    c_spider        = mock_task('c_spider',        'fuzz-test',      '200m', 256, 10)

    # ── 依赖关系（安全扫描串行，工具和 Fuzz 并行） ──
    secret_scan >> sudo_check >> crypto_check >> data_check >> port_scan >> binary_scan >> virus_scan
    virus_scan >> [auto_tag, mail_notify, xsan_sync, nmap_scan, dtfuzz, c_spider]
```

#### 14.5.2 创建 DAG

```bash
# 在 mon-af namespace 创建 ConfigMap（模拟 gitSync）
kubectl -n mon-af create configmap airflow-dags \
  --from-file=dag_mock_workflow.py

# 或直接复制到 Airflow 容器的 dags 目录
kubectl -n mon-af exec deploy/airflow-webserver -- \
  bash -c "echo '$(cat dag_mock_workflow.py | base64)' | base64 -d > /opt/airflow/dags/dag_mock_workflow.py"
```

### 14.6 SmartIDE（WebIDE 开发环境）

#### 14.6.1 概述

SmartIDE 是一个开源的容器化 WebIDE 方案，通过 K8s Deployment 部署 VS Code Server，浏览器即可访问完整开发环境。正式方案中 WebIDE 组用 5 台 32u/64G 节点，验证环境简化为 1 个 Pod 验证流程。

#### 14.6.2 部署 SmartIDE

```bash
# 安装 SmartIDE CLI
curl -fsSL https://smartide.cn/install.sh | bash

# 或直接用 Helm 部署（K8s 原生方式）
kubectl create namespace webide

kubectl -n webide apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: smartide
  labels:
    app: smartide
spec:
  replicas: 1
  selector:
    matchLabels:
      app: smartide
  template:
    metadata:
      labels:
        app: smartide
    spec:
      nodeSelector:
        workload: mon-af       # 验证环境复用 mon-af 标签
      containers:
      - name: smartide
        image: smartide/smartide:latest
        ports:
        - containerPort: 8080
          name: web
        - containerPort: 22
          name: ssh
        env:
        - name: WORKSPACE_DIR
          value: /home/project
        resources:
          requests: { cpu: "500m", memory: "1Gi" }
          limits:   { cpu: "1u",   memory: "2Gi" }
        readinessProbe:
          httpGet: { path: /, port: 8080 }
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet: { path: /, port: 8080 }
          initialDelaySeconds: 60
          periodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: smartide
spec:
  type: NodePort
  selector:
    app: smartide
  ports:
  - name: web
    port: 8080
    nodePort: 30080
  - name: ssh
    port: 22
    nodePort: 30022
EOF

# 等待就绪
kubectl -n webide wait --for=condition=ready pod -l app=smartide --timeout=120s
```

#### 14.6.3 验证 WebIDE

```bash
# 获取访问地址
echo "VS Code: http://$(kubectl get node -o jsonpath='{.items[0].status.addresses[0].address}'):30080"

# 浏览器打开 → 进入 VS Code Server 界面
# 可在终端中执行命令验证开发环境
```

#### 14.6.4 验证清单

| # | 验证项 | 预期结果 |
|:--|:-----|:--------|
| 1 | Pod 调度到指定节点 | `kubectl get pod -n webide -o wide` 显示节点名 |
| 2 | 浏览器可打开 VS Code | 页面加载正常 |
| 3 | 终端可用 | 在 VS Code 终端执行 `uname -a` 有输出 |
| 4 | Pod 日志正常 | `kubectl -n webide logs deploy/smartide` 无报错 |
| 5 | 删除 Pod 后重建可用 | `kubectl -n webide delete pod -l app=smartide` 自动恢复 |

### 14.7 监控部署（验证版）

#### 14.7.1 Prometheus

```bash
# 创建 prometheus.yml 配置
kubectl -n mon-af create configmap prometheus-config \
  --from-literal=prometheus.yml='
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: airflow
    static_configs:
      - targets: ["airflow-statsd:9102"]
    metrics_path: /metrics

  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
'

# 部署 Prometheus
kubectl -n mon-af apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      nodeSelector:
        workload: mon-af
      containers:
      - name: prometheus
        image: prom/prometheus:v2.50.0
        args:
        - '--storage.tsdb.retention.time=1d'
        - '--config.file=/etc/prometheus/prometheus.yml'
        ports:
        - containerPort: 9090
        resources:
          requests: { cpu: "200m", memory: "256Mi" }
          limits:   { cpu: "500m", memory: "512Mi" }
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus
      volumes:
      - name: config
        configMap:
          name: prometheus-config
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
spec:
  type: NodePort
  selector:
    app: prometheus
  ports:
  - port: 9090
    nodePort: 30090
EOF
```

#### 14.7.2 Grafana

```bash
kubectl -n mon-af apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      nodeSelector:
        workload: mon-af
      containers:
      - name: grafana
        image: grafana/grafana:10.2.0
        env:
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "true"
        ports:
        - containerPort: 3000
        resources:
          requests: { cpu: "100m", memory: "128Mi" }
          limits:   { cpu: "200m", memory: "256Mi" }
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
spec:
  type: NodePort
  selector:
    app: grafana
  ports:
  - port: 3000
    nodePort: 30300
EOF
```

#### 14.7.3 AlertManager

```bash
kubectl -n mon-af apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      labels:
        app: alertmanager
    spec:
      nodeSelector:
        workload: mon-af
      containers:
      - name: alertmanager
        image: prom/alertmanager:v0.26.0
        ports:
        - containerPort: 9093
        resources:
          requests: { cpu: "50m", memory: "64Mi" }
          limits:   { cpu: "100m", memory: "128Mi" }
---
apiVersion: v1
kind: Service
metadata:
  name: alertmanager
spec:
  type: NodePort
  selector:
    app: alertmanager
  ports:
  - port: 9093
    nodePort: 30903
EOF
```

### 14.8 验证步骤

#### 14.8.1 基础检查

```bash
# 1. 所有组件就绪
kubectl get pods -n mon-af

# 期望输出：
# airflow-webserver-xxx    1/1 Running
# airflow-scheduler-xxx    1/1 Running
# airflow-worker-xxx       1/1 Running
# airflow-triggerer-xxx    1/1 Running
# airflow-postgresql-0     1/1 Running
# prometheus-xxx           1/1 Running
# grafana-xxx              1/1 Running
# alertmanager-xxx         1/1 Running

# 2. 节点标签
kubectl get nodes --show-labels | grep workload
```

#### 14.8.2 手动触发 DAG

```bash
# 方法 1：Web UI 手动触发
# 打开 Airflow Web UI → 找到 daily_mock_workflow → Trigger DAG

# 方法 2：CLI 触发
kubectl -n mon-af exec deploy/airflow-scheduler -- \
  airflow dags trigger daily_mock_workflow
```

#### 14.8.3 观察 Pod 调度过程

```bash
# 实时观察 Pod 创建销毁
watch -n 1 'kubectl get pods -A | grep -E "secret_scan|sudo_check|crypto|data_check|port_scan|binary_scan|virus_scan|auto_tag|mail|xsan|nmap|dtfuzz|c_spider"'

# 观察单个 Pod 日志
kubectl -n security-scan logs -l app=secret_scan -f

# 验证 Pod 落在正确节点
kubectl get pod -o wide -n security-scan
```

#### 14.8.4 检查清单

| # | 验证项 | 预期结果 | 通过 |
|:--|:-----|:--------|:--:|
| 1 | Airflow Web UI 可访问 | 页面正常打开 | ☐ |
| 2 | DAG 列表中显示 `daily_mock_workflow` | 状态 `active` | ☐ |
| 3 | 手动 Trigger 后 Task 开始调度 | Pod 状态 `Running` | ☐ |
| 4 | 7 项安全扫描按依赖串行 | 一次只运行 1 个扫描 Pod | ☐ |
| 5 | 工具服务和 Fuzz 并行执行 | 6 个 Pod 同时 Running | ☐ |
| 6 | 每个 Pod 日志输出任务名 + SUCCESS | `Result: SUCCESS` | ☐ |
| 7 | Pod 跑完自动销毁 | `is_delete_operator_pod=true` 生效 | ☐ |
| 8 | SmartIDE 浏览器可访问 | VS Code 界面正常 | ☐ |
| 9 | Grafana 可访问（http://NODE:30300） | Dashboard 正常 | ☐ |
| 10 | Prometheus 采集到 Airflow metrics | `airflow_dag_*` 指标有值 | ☐ |
| 11 | AlertManager 可访问 | http://NODE:30903 | ☐ |

#### 14.8.5 模拟失败告警

```python
# 在 DAG 中添加一个会失败的任务
fail_task = KubernetesPodOperator(
    task_id='test_fail_task',
    namespace='security-scan',
    image='busybox',
    cmds=['sh', '-c', 'echo "FAILING!" && exit 1'],
    node_selector={'workload': 'security-scan'},
    is_delete_operator_pod=True,
)
```

1. 触发该 DAG → Task 失败 → Airflow UI 显示红色
2. 验证 Prometheus 指标 `airflow_task_fail_count` 增加

### 14.9 访问地址汇总

| 服务 | 地址 |
|:-----|:-----|
| SmartIDE (VS Code) | `http://<NODE_IP>:30080` |
| Airflow Web UI | `http://<NODE_IP>:<NodePort>` |
| Prometheus | `http://<NODE_IP>:30090` |
| Grafana | `http://<NODE_IP>:30300` |
| AlertManager | `http://<NODE_IP>:30903` |

### 14.10 清理

```bash
# 删除所有验证资源
kubectl delete ns security-scan tools fuzz-test mon-af webide
kubectl label node $(kubectl get nodes -o name) workload-
```

### 14.11 与正式方案映射

| 验证环境 | 正式环境 |
|:--------|:--------|
| 单节点 + 全标签 | 2 节点 airflow 独享 + 各自标签 |
| `mock-task:latest` 伪代码 | 正式扫描 / Fuzz / 工具镜像 |
| Airflow 1 副本 | Airflow 2 副本 HA |
| Prom 1 副本 | 不变（单点） |
| 无 Loki | 华为云 LTS |
| 无 Jenkins / MySQL / WebIDE | 正常部署 |
