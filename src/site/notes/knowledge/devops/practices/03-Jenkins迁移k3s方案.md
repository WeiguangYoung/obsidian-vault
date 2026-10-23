---
{"dg-publish":true,"permalink":"/knowledge/devops/practices/03-Jenkins迁移k3s方案/","dg-note-properties":{}}
---


# Jenkins 迁移 k3s 方案（docker-compose → k3s）

> 现状：master 跑在腾讯云宿主机 docker-compose（`jenkins/jenkins:lts` 2.555.3，`/root/jenkins_home` 583M，端口 8080/50000，内存限额 1G，挂载 docker.sock 供 pipeline 构建）
> 目标：迁到 k3s 集群（server 杭州 `203.0.113.11` + agent 上海 `203.0.113.13`，2 节点 4C/3.5G）
> 状态：**方案待评审，未执行**。集群侧 Jenkins k8s agent 链路（插件/SA/token/CA）已就绪，见 Jenkins部署实战.md 第七章

---

## 〇、先回答"值不值得迁"（三个前置拷问）

| # | 问题 | 事实 | 结论 |
|---|---|---|---|
| 1 | 腾讯云跑得好好的，为什么迁？ | master 在本机资源充裕，k3s 两台阿里云合计才 4C/3.5G | **纯资源角度不值得**。合理的迁移动机：①实践 K8s 有状态应用运维（学习价值）②腾讯云机退役/降本 ③master 与 k8s 动态 agent 同集群收口 |
| 2 | docker.sock 怎么办？ | 当前 pipeline 依赖宿主机 docker 构建镜像；**k3s 运行时是 containerd，没有 docker.sock** | 最大功能损失点，见坑 A |
| 3 | 内存够吗？ | 杭州 server：k3s+APISIX+Cloudreve 已占 ~780M/1.7G；上海 agent：空机 available ~1.6G | master（Java 堆+限额 1G）**只能钉上海节点**，钉杭州必 OOM |

---

## 一、目标架构

```
浏览器/Git webhook ──> http://203.0.113.13:30080   (NodePort, 上海节点)
                          │
                 ┌────────▼─────────┐
                 │  Jenkins master  │  Deployment(1副本) 钉在上海节点
                 │  jenkins:lts     │  hostPath /data/jenkins_home
                 └───┬──────────┬───┘
        JNLP/WebSocket │          │ K8s API (Cloud 配置)
                 ┌───▼───┐  ┌────▼──────────────┐
   win-agent-01 ─┤ 公网   │  │ k8s 动态 agent pod │ 同集群走 svc 内连
   外部 agent ───┘30500/WS│  └───────────────────┘
                          │
            集群: 杭州 server(控制面) + 上海 agent(工作负载)
```

- **namespace**：`jenkins`（已建，SA/RBAC/token 均为 agent 线备好）
- **master 调度**：`nodeSelector: kubernetes.io/hostname=i-xxxxxxxx0002`（上海）
- **存储**：**hostPath**（不用 PVC——master 单实例绑死单机，RWO PVC 经 local-path 绕一圈无收益；数据就在 `/data/jenkins_home`，节点上直接看得见，备份 rsync 都简单）
- **对外**：Web NodePort **30080**；agent 回连二选一：TCP **30500**→50000，或全切 WebSocket（只开 30080 单端口，win-agent/agent-01 都友好）

---

## 二、迁移前必须做的决定（三项，先拍板后动手）

### 决定 1 · docker 构建功能（坑 A：k3s 没有 docker.sock）

现有 job 里凡是 pipeline 中 `docker build/run` 的，迁移后**全部报错**。选项：

| 方案 | 说明 | 适用 |
|---|---|---|
| **B1 构建下沉到 agent（推荐）** | master 只做调度，构建交给现有节点：k8s agent pod（kaniko 或挂不到 docker 的纯编译）、win-agent-01、agent-01 | 与"动态 agent"路线一致，master 变轻 |
| B2 kaniko 替换 docker build | 无守护进程构建镜像：`gcr.io/kaniko-project/executor` 在 pod 里 build+push 到 Harbor/镜像仓库 | 专门迁"打镜像"类 job |
| B3 dind 特权 sidecar | pod 里跑 dockerd | ❌ 小内存集群禁，安全面也差 |
| B4 不迁，master 留腾讯云 | 反向选项：迁不动就不迁，只把 k8s 当 agent 池（现状即如此） | 保守方案 |

**迁移前动作**：盘点 `grep -rl 'docker' $JENKINS_HOME/jobs/*/config.xml`，列出依赖 docker 的 job 清单，逐个定 B1/B2。

### 决定 2 · 公网 IP 变更的连锁反应（坑 B）

`203.0.113.12` → `203.0.113.13:30080`，全部要改：

- [ ] win-agent-01 / intranet-agent-01 启动命令里的 `-url`（MSI 装的改服务配置或重跑向导）
- [ ] Git 平台 webhook（Gitee/GitHub 里指向 `…:8080/github-webhook/` 的地址）
- [ ] 邮件通知里的构建链接（Jenkins URL 全局配置改）
- [ ] 所有收藏/文档/脚本里硬编码的 master 地址
- [ ] 上海机安全组放行 30080（+30500 若走 TCP JNLP）；**杭州机原 6443 规则不动**
- [ ] （可选）域名 + 反代收口，以后换 IP 不再伤筋动骨

### 决定 3 · kubernetes-plugin 的 URL 双层配置（坑 C：不做就是 hairpin 大坑）

- **全局 Jenkins URL** 改 `http://203.0.113.13:30080`（浏览器、外部 agent 用）
- **Cloud 配置里单独填 Jenkins URL** = `http://jenkins-master.jenkins.svc.cluster.local:8080` → k8s 动态 agent pod 走集群内 service 回连 master，**不出集群不绕公网**
- Cloud 的 Kubernetes server URL 用 `https://kubernetes.default.svc` + 凭证类型选 **Kubernetes Service Account**（pod 自带 SA token，不再需要 /root/ci-secrets 那个外部 token——那是"master 在集群外"时代产物）

---

## 三、K8s 资源清单（可直接 apply）

### 3.1 master 的 RBAC（比外部 agent 那套大：要建 pod）

```yaml
apiVersion: v1
kind: ServiceAccount
metadata: { name: jenkins-master, namespace: jenkins }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: jenkins-master, namespace: jenkins }
rules:
- apiGroups: [""]
  resources: [pods, pods/exec, persistentvolumeclaims]
  verbs: [create, delete, get, list, patch, watch]
- apiGroups: [""]
  resources: [events]
  verbs: [create, watch]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: jenkins-master, namespace: jenkins }
subjects: [{ kind: ServiceAccount, name: jenkins-master, namespace: jenkins }]
roleRef: { kind: Role, name: jenkins-master, apiGroup: rbac.authorization.k8s.io }
```

### 3.2 Deployment + Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: jenkins-master, namespace: jenkins }
spec:
  replicas: 1
  strategy: { type: Recreate }          # 有状态：严禁 RollingUpdate 双挂 hostPath
  selector: { matchLabels: { app: jenkins-master } }
  template:
    metadata: { labels: { app: jenkins-master } }
    spec:
      serviceAccountName: jenkins-master
      nodeSelector: { kubernetes.io/hostname: i-xxxxxxxx0002 }   # 上海，唯一放得下的节点
      containers:
      - name: jenkins
        image: docker.io/jenkins/jenkins:lts        # 两节点都已配 DaoCloud 加速
        args: ["--httpPort=8080"]
        env:
        - { name: JAVA_OPTS, value: "-Xmx768m -Duser.timezone=Asia/Shanghai" }
        - { name: JENKINS_SLAVE_AGENT_PORT, value: "50000" }
        ports:
        - { containerPort: 8080, name: http }
        - { containerPort: 50000, name: jnlp }
        resources:
          requests: { cpu: 200m, memory: 768Mi }
          limits:   { cpu: "1",  memory: 1Gi }
        readinessProbe: { httpGet: { path: /login, port: 8080 }, initialDelaySeconds: 40, periodSeconds: 10 }
        volumeMounts:
        - { name: home, mountPath: /var/jenkins_home }
        - { name: tmp, mountPath: /tmp }
      volumes:
      - { name: home, hostPath: { path: /data/jenkins_home, type: Directory } }
      - name: tmp
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata: { name: jenkins-master, namespace: jenkins }
spec:
  type: NodePort
  selector: { app: jenkins-master }
  ports:
  - { name: http, port: 8080, targetPort: 8080, nodePort: 30080 }
  - { name: jnlp, port: 50000, targetPort: 50000, nodePort: 30500 }
```

要点：
- **uid 对齐**：容器内 jenkins 用户 uid=1000，hostPath 目录在节点上 `chown -R 1000:1000 /data/jenkins_home`，否则起不来（Permission denied 写 secrets）
- **不再挂 docker.sock / docker CLI / group_add**（对应决定 1，B1/B2 路线下 master pod 无 docker）
- 原 compose 的 `-Djenkins.model.Jenkins.slaveAgentPort=50000` 参数由 env `JENKINS_SLAVE_AGENT_PORT` 等价承担
- 镜像 tag 建议迁移时从 `lts` 钉到 `lts-2.555.3`，避免"迁移+升级"双变量

---

## 四、实施步骤（维护窗口约 30 分钟）

```
T-1 天    决定 1/2/3 拍板；docker 依赖 job 清单盘点完
T-10min   上海节点准备：mkdir -p /data/jenkins_home
T-0       ① 腾讯云旧 master：docker stop jenkins（保留容器不删 = 回滚保险）
同步      ② rsync -aHAX --delete /root/jenkins_home/ root@203.0.113.13:/data/jenkins_home/
              （583M 走公网约 2-5 分钟；可提前热备一次缩小增量）
          ③ chown -R 1000:1000 /data/jenkins_home
部署      ④ kubectl apply -f 3.1 + 3.2 → kubectl -n jenkins rollout status deploy/jenkins-master
验证      ⑤ 新地址登录：插件齐/凭据在/job 列表完整/构建历史可看
          ⑥ 系统配置：Jenkins URL=新地址；JNLP 协议/端口按需
          ⑦ 跑一个老 job（不含 docker 的）+ 跑一个 agent 调度（先 win-agent-01）
切流      ⑧ webhook 改新地址；两台外部 agent 改 -url 重启
观察      ⑨ ≥1 周旧容器保持 stopped 可 docker start 即回滚
退役      ⑩ 稳定后腾讯云容器 down、jenkins_home 打最终备份留档
```

---

## 五、风险与缓解

| 风险 | 概率 | 缓解 |
|---|---|---|
| hostPath 权限/属主不对，pod 起不来 | 高（首次必撞） | 步骤③ 强制 chown；journal 里 `java.io.IOException: Permission denied` 即此 |
| docker 依赖 job 迁移后批量失败 | 高（不处理必发生） | 决定 1 先盘点；失败 job 逐个 B1/B2 改造，不阻塞整体切换 |
| 上海节点挂 = master 挂（单点） | 低 | 接受（实验集群）；缓解=每日 rsync /data/jenkins_home 回腾讯云冷备（cron 一条命令） |
| 内存挤兑：master 1G + agent 常驻 ~300M + 系统 ~400M ≈ 上海机 1.8G 上限 | 中 | Xmx 压 768M + 2G swap 兜底 + limits 硬顶；监控 `free` 与 `kubectl top node` |
| 公网 30080 弱防护（首次装会暴露 setup 页？不会——home 里有 unlock key，但公网爆破存在） | 中 | 迁移后立刻：强密码/禁用匿名 API（已 403 ✅）/考虑 fail2ban；根治走域名+HTTPS+反代 |
| agent 隧道抖动（跨 VPC 回落 6443，remotedialer timeout 每 35s） | 低 | 属预期（k3s部署实战.md 坑注），影响仅日志；在意就 VPC 对等连接 |

---

## 六、回滚预案

任一环节卡死 >30 分钟即回滚，成本极低：
1. 腾讯云 `docker start jenkins`（数据从未删改，rsync 单向）
2. agent 改回 `-url http://203.0.113.12:8080/` 重启；webhook 改回
3. k3s 侧 `kubectl delete -f` 清场，/data/jenkins_home 留着下次分析
4. 注意：**窗口内若已在新 master 上产生新构建/改配置，回滚会丢这段增量**——回滚前先把新 home rsync 回来覆盖

---

## 七、迁移完成后的红利（验收标准）

- [ ] master 由 Deployment 管理：`kubectl rollout restart` 即重启，配置声明式可 git 化
- [ ] k8s 动态 agent 闭环：Cloud=in-cluster SA，agent pod 走 svc 内连，构建完 idle 5min 自动回收
- [ ] 三平台执行池成型：Linux(k8s pod) / Windows(win-agent-01) / 内网(agent-01)
- [ ] master 每日冷备 cron（rsync 回腾讯云）跑通一次恢复演练
- [ ] Jenkins部署实战.md 第七章（集群外 master 配法）标注"被本方案取代"

相关文档：
- [[knowledge/devops/jenkins/02-Jenkins部署实战\|Jenkins部署实战]]
- [[notes/1-工作/k3s部署实战\|k3s部署实战]]
