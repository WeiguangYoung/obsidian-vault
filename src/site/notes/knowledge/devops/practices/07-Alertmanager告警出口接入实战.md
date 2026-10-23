---
{"dg-publish":true,"permalink":"/knowledge/devops/practices/07-Alertmanager告警出口接入实战/","tags":["监控","告警","Alertmanager","企业微信","k3s","KPS","实战"],"dg-note-properties":{"date":"2026-09-27","updated":"2026-09-28","tags":["监控","告警","Alertmanager","企业微信","k3s","KPS","实战"]}}
---


# Alertmanager 告警出口接入实战（企微群机器人）

> 背景：k3s 上 kube-prometheus-stack 早已部署，Prometheus 规则能产告警、Alertmanager 也收到了，
> 但 **Alertmanager 一直是空配置（无 receiver）→ 告警发不出去**。本文记录 2026-09-27 把它打通的全过程。
> 环境：非 helm release 的监控栈（`helm template + kubectl apply` 管的，`helm status kps` = release not found）。

---

## 一、问题定位

| 环节 | 状态（改造前） |
|---|---|
| Prometheus 规则评估 → 产告警 | ✅ 正常（`ALERTS{alertstate="firing"}` 有 9 条） |
| Prometheus → Alertmanager 接入 | ✅ `spec.alerting` 已 patch |
| **Alertmanager → 通知出口** | ❌ **`route.receiver = None`、`receivers = []`** |

**根因**：Alertmanager CR 从没挂 `configSecret`，operator 用默认空配置生成了 `*-generated` secret → 无 receiver → 告警进 AM 后无处可去。

---

## 二、架构设计（为什么需要一个转发器）

```
Prometheus 规则 → Alertmanager(Pod, home-node 家宽)
                      │  webhook_configs
                      ▼
              http://10.88.0.3:9099/        ← 上海节点上的转发器
                      │  POST 企微 API
                      ▼
        企业微信群机器人（qyapi.weixin.qq.com）
```

**为什么不让 Alertmanager 直接打企微？**
1. **格式适配**：Alertmanager 的 webhook 是**通用 JSON**（`{status, alerts:[{labels,annotations}]}`），而企微群机器人要 `{msgtype:"text", text:{content}}` → 必须有中转做转换。
2. **网络/可测性**：中转放上海节点（常驻、wg 可达 `10.88.0.3`），AM 在 home-node 只能经 overlay 访问 → 走 WG 平面稳定。

**为什么转发器放上海节点（10.88.0.3）而不是 home-node？**
- 上海节点是常驻云机（22 天 uptime、负载 0.00、有公网出网），**不依赖家宽**；
- AM 在 home-node，经 WG `10.88.0.0/24` 到 `10.88.0.3:9099` 直达；
- 转发器无状态、零依赖（纯 stdlib python3），挂了 systemd `Restart=always` 5s 自愈。

---

## 三、落地步骤

### 3.1 转发器（上海节点 203.0.113.13）

脚本：`/opt/wa-relay/relay.py`（纯 stdlib：`urllib` + `http.server`）
```python
#!/usr/bin/env python3
"""Alertmanager -> 企业微信群机器人 转发器 (stdlib only)."""
import json, os, sys, datetime, urllib.request
from http.server import BaseHTTPRequestHandler, HTTPServer

WEBHOOK = os.environ.get("WECHAT_WEBHOOK", "")
LISTEN_HOST = os.environ.get("LISTEN_HOST", "0.0.0.0")
LISTEN_PORT = int(os.environ.get("LISTEN_PORT", "9099"))
MAXLEN = 1800  # 企微 text 单条上限约 2048 字节，留余量

def post_wechat(content: str):
    body = json.dumps({"msgtype": "text", "text": {"content": content[:MAXLEN]}}).encode()
    req = urllib.request.Request(WEBHOOK, data=body, headers={"Content-Type": "application/json"})
    with urllib.request.urlopen(req, timeout=10) as r:
        return r.read().decode()

def fmt(payload: dict) -> str:
    status = payload.get("status", "?")
    alerts = payload.get("alerts", [])
    icon = {"firing": "🔥", "resolved": "✅"}.get(status, "🔔")
    lines = [f"{icon} [{status.upper()}] {len(alerts)} 条告警"]
    for a in alerts[:10]:
        lab = a.get("labels", {}) or {}
        ann = a.get("annotations", {}) or {}
        lines.append(f"\n▸ {lab.get('alertname','?')} ({lab.get('severity','-')})")
        if lab.get("instance"): lines.append(f"  实例: {lab['instance']}")
        if ann.get("summary"): lines.append(f"  摘要: {ann['summary']}")
        if ann.get("description"): lines.append(f"  详情: {ann['description']}")
    if len(alerts) > 10: lines.append(f"\n… 其余 {len(alerts)-10} 条省略")
    lines.append(f"\n时间: {datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    return "\n".join(lines)

class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        n = int(self.headers.get("Content-Length", 0))
        raw = self.rfile.read(n)
        try:
            payload = json.loads(raw.decode() or "{}")
            res = post_wechat(fmt(payload))
            print("relayed:", res, file=sys.stderr, flush=True)
            code, out = 200, b"ok"
        except Exception as e:
            print("relay error:", repr(e), file=sys.stderr, flush=True)
            code, out = 500, b"err"
        self.send_response(code); self.send_header("Content-Length", str(len(out))); self.end_headers()
        self.wfile.write(out)
    def do_GET(self):
        self.send_response(200); self.send_header("Content-Length", "2"); self.end_headers(); self.wfile.write(b"ok")
    def log_message(self, *a): pass

if __name__ == "__main__":
    if not WEBHOOK: sys.exit("WECHAT_WEBHOOK required")
    print(f"listening {LISTEN_HOST}:{LISTEN_PORT}", flush=True)
    HTTPServer((LISTEN_HOST, LISTEN_PORT), Handler).serve_forever()
```

env：`/etc/wa-relay.env`（`chmod 600`）
```
WECHAT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=***
LISTEN_HOST=0.0.0.0
LISTEN_PORT=9099
```

systemd：`/etc/systemd/system/wa-relay.service`
```ini
[Unit]
Description=Alertmanager -> WeCom webhook relay
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
EnvironmentFile=/etc/wa-relay.env
ExecStart=/usr/bin/python3 /opt/wa-relay/relay.py
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
```
```bash
systemctl daemon-reload && systemctl enable --now wa-relay
```

自测：
```bash
curl -s -X POST http://127.0.0.1:9099/ -H 'Content-Type: application/json' \
  -d '{"status":"firing","alerts":[{"labels":{"alertname":"TestAlert","severity":"warning","instance":"10.88.0.1:9100"},"annotations":{"summary":"转发器链路自测"}}]}'
# → ok；journalctl -u wa-relay → relayed: {"errcode":0,"errmsg":"ok"}
```

### 3.2 Alertmanager 配置（关键：挂 configSecret）

> ⚠️ **这套监控栈没有 helm release，`helm upgrade` 用不了**（会报 `has no deployed releases`）→ 只能改 **Alertmanager CR 的 configSecret**，让 operator 渲染。

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
# 1) 建 base secret（名称 = <alertmanager名>，非必带 -generated；-generated 由 operator 渲染产出）
kubectl -n monitoring create secret generic alertmanager-kps-alertmanager \
  --from-file=alertmanager.yaml=/tmp/am.yaml --dry-run=client -o yaml | kubectl apply -f -
# 2) 给 CR 挂 configSecret
kubectl -n monitoring patch alertmanager kps-alertmanager --type=merge \
  -p '{"spec":{"configSecret":"alertmanager-kps-alertmanager"}}'
# 3) operator reconcile（~20s）→ 渲染进 alertmanager-kps-alertmanager-generated
```

`/tmp/am.yaml`：
```yaml
global:
  resolve_timeout: 5m

route:
  receiver: wecom
  group_by: ['alertname', 'instance']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
  - receiver: blackhole          # Watchdog / InfoInhibitor 是常驻告警，去噪
    matchers:
    - alertname =~ "Watchdog|InfoInhibitor"

receivers:
- name: wecom
  webhook_configs:
  - url: http://10.88.0.3:9099/
    send_resolved: true
- name: blackhole
```

**验证渲染生效**（`/api/v2/status` 在此版本不回传 config，别看它）：
```bash
kubectl -n monitoring get secret alertmanager-kps-alertmanager-generated \
  -o jsonpath='{.data.alertmanager\.yaml\.gz}' | base64 -d | gunzip
kubectl -n monitoring exec alertmanager-kps-alertmanager-0 -c alertmanager -- \
  cat /etc/alertmanager/config_out/alertmanager.env.yaml
```

### 3.3 补核心告警规则（PrometheusRule）

> KPS 自带 125 条默认规则（TargetDown / CrashLoop / NodeNotReady / 磁盘将满…）已经很全，缺的是**项目化规则**——尤其 Jenkins 构建质量（平安 JD 点名项）。

**怎么加**：`PrometheusRule` **必须带 `release: kps` 标签**（Prometheus CR 的 `ruleSelector: {matchLabels: {release: kps}}` 才选得中），否则 Just 创建、不加载。

```bash
kubectl apply -f /tmp/project-rules.yaml   # 见下
# operator reconcile（~30s）→ 渲染进 configmap，config-reloader reload
```

`kps-project-alerts` 规则（5 条）：

| 组 | 规则 | 表达式 | 级别 |
|---|---|---|---|
| project.jenkins | JenkinsBuildFailed | `default_jenkins_builds_last_build_result_ordinal > 0` | critical |
| project.jenkins | JenkinsJobUnhealthy | `default_jenkins_builds_health_score < 50` | warning |
| project.jenkins | JenkinsTargetDown | `up{job="jenkins"} == 0` | critical |
| project.node | NodeMemoryAvailableLow | `node_memory_MemAvailable_bytes < 300Mi` | warning |
| project.node | NodeMemoryAvailableCritical | `node_memory_MemAvailable_bytes < 150Mi` | critical |

**Jenkins 健康分算法**（实测反推，见「五、坑 7」）：最近 ≤5 次构建平均分，SUCCESS=100 / FAILURE·ABORTED=0。

**验证加载**（看 `/api/v1/rules` 里 `project.*` 组 `health=ok`）：
```bash
curl -s -u admin:*** http://<grafana>/api/datasources/proxy/uid/prometheus/api/v1/rules?type=alert
```

---

## 四、验收

| 检查项 | 结果 |
|---|---|
| 转发器自测 | `errcode:0` ✅ |
| AM 渲染配置 | 含 `receiver: wecom` + `webhook_configs.url: http://10.88.0.3:9099/` ✅ |
| **端到端真实投递** | relay 在 AM 加载后 **33s** 收到投递（= group_wait 30s）→ `relayed: errcode 0` ✅ |
| **核心规则实测** | `JenkinsJobUnhealthy`（sherpa-lite/main 健康分 40）真实 firing → 推企微群 ✅ |
| 企微群 | 收到测试 + 真实告警消息 ✅ |
| 去噪 | `Watchdog` 走 blackhole，不再刷屏 ✅ |

---

## 五、关键坑与认知

1. **非 helm release 的监控栈改配置**：只能 `patch Alertmanager CR` + 建 base secret，**不能 `helm upgrade`**。CR 里 `configSecret` 不设 → operator 用默认空配置（这就是"AM 收到告警但发不出去"的根因）。
2. **secret 命名**：base secret 名 = CR 的 `configSecret` 值；渲染产物是 `<alertmanager名>-generated`。改 base secret 后 operator 会在约 20s 内重新渲染。
3. **`/api/v2/status` 的 config 可能为空**（该 AM 版本行为）→ 验证要看 pod 内 `/etc/alertmanager/config_out/alertmanager.env.yaml` 或 `-generated` secret，别被 API 误导。
4. **Alertmanager webhook 是通用 JSON**，企微/钉钉要专有格式 → **必须有中转**；除非直接用 Prometheus 的 `webhook_configs` 出站插件。
5. **`group_wait: 30s`** 是首条告警的首次投递延迟，属正常（批量聚合），测试时别以为没通。
6. **Watchdog 必删/屏蔽**：它是 Prometheus 自带的"死信开关"（永远 firing），不屏蔽会跟真实告警一起推、稀释信噪比。
7. **PrometheusRule 两个坑（2026-09-27）**：
   - **`default()` 不是 Prometheus 模板函数**（那是 Grafana/Jsonnet 生态的）→ 写了 admission webhook 直接拒收 `PrometheusRule`。用 `$labels.instance` 代替。
   - **必须带 `release: kps` 标签**，否则 operator 不选它、规则静静躺在那里不加载。
   - "加了不生效"别急：operator reconcile ~30s + config-reloader reload；别在第一眼就判死刑。
8. **告警阈值调整的坑：删数组项必须 `replace` 不是 `apply`（2026-09-28）**
   - `kubectl apply` 三方合并对数组取并集 → **文件里删掉的规则会从 live 合回来**，删不掉。
   - 改用 **`kubectl replace -f`** 整份覆盖才行。
   - 规律：**只要是用脚本生成整份 YAML 去增删「数组元素」，就用 `replace`**；`replace` 要求资源已存在，改前先备份。

9. **转发器单线程会被「半开连接」永久堵死（2026-09-28 真实故障）**：
   - 现象：告警在 AM 里正常 firing，但**企微一条都收不到**；AM 日志 `dial tcp 10.88.0.3:9099: connect: connection timed out` 连环重试失败。
   - 根因：relay 用**单线程 `HTTPServer` + 无 socket 超时**，一个慢连接/端口扫描半开连接（日志里 `198.51.100.14` 的 `ConnectionResetError`）就能把 server 读请求的循环卡死 → accept 队列堆满（`ss` 里 `Recv-Q > Send-Q`）→ 所有新 POST 超时。
   - **陷阱：`systemctl is-active` 报 active**（进程还在 `Ss` 态），但实际已不干活。**判活要探端口/发请求，或看 `ss -ltn` 的 Recv-Q 是否堆积**。
   - 修复（已落地，`deploy/wa-relay/relay.py` + 远端 `/opt/wa-relay/relay.py`）：① `ThreadingMixIn`+`daemon_threads` 线程化；② `Handler.timeout=20` 断开半开连接；③ 出站 `urlopen(timeout=10)`。远端 Python 3.6.8 兼容。
   - 验证：端到端 `ok`；**抗阻塞压测**（半开连接占住时正常 GET/POST 仍 200）；加固后 AM 发送失败 0 次。

---

## 五之二、配套运维事故与调优（2026-09-28）

### A. 告警「误报」自查：先怀疑取证方法，别急着改配置

- 上午排查「Jenkins 宿主磁盘告警进入 pending」，我一度通过 SSH 截图 `curl /api/v2/status` 看 AM 配置，结论写成「`blackhole` 子路由无条件匹配 → 吞掉所有告警 → 企微收不到」。
- **实际是错的**：那次只打印了配置原文**最后 3000 字符**再 `grep` 出 `route:` 几行，子路由的 `matchers`（在截断位置之前）被 grep 过滤掉了 → 看起来「无匹配条件」，实际有。
- 用户随后贴出真实收到的 `🔥 [FIRING] JenkinsHostDiskLow`（17:34:36），链路完全正常：规则 17:34:06 转 firing → `group_wait 30s` → 17:34:36 投出。
- **两个教训**：
  1. **pending 状态的告警根本不会进 Alertmanager**——查 AM 里没有它 ≠ 被吞；要查 Prometheus `/api/v1/rules` 的 `state`。
  2. **取证要取全，别用「截断 + grep」拼结论**。看完整 route 配置应直接读 `-generated` secret 或 pod 内 `config_out`。

### B. 本机磁盘告急 → 清理（阈值 20% 那条真触发）

- 事故：本机 `/` 83%（可用 8.9G），触发 `JenkinsHostDiskLow`。
- 找到元凶：`/root/disk-alert-sim.bin`（6.0G，磁盘告警模拟文件，用户测试产物）。
- 删除后：42G→36G，可用 8.9G→15G，**83%→71%**。告警下个周期自动 resolve。
- 备注：`/root/.cache`（2.9G，含 playwright）等其它大户未动，按需再说。

### C. 内存调优：清 Chromium 常驻进程

- 现象：2G 小机，`used 1.5G / avail 369M`，**swap 已用 1.9G**（长期紧张）。
- 占用 Top：`openclaw gateway`（node）666M / Jenkins(java) 204M / YDService 79M / dockerd+rclone ~40M，外加 **Playwright Chromium 12 个常驻进程 ~100M**。
- 处置：`pkill -f chrome-linux64/chrome` + `pkill -f chrome-devtools-mcp` 杀掉整棵 Chromium 树（12→0）。
  - **注意**：`browser stop` 对「持有主会话的 attachOnly 浏览器」无效（返回 `running:true`），只能直接 kill 进程。
  - 收益：avail **369M→438M**，**swap 1.9G→1.6G**（进程死后换出页释放，比预期收益更大）。
- **可恢复性**：Chromium 只是杀进程，**没删文件**——可执行文件 + `userDataDir` 都在，下次用浏览器会自动拉起（登录态若在持久化 userDataDir 里则保留）。
- 未做（待定）：Jenkins JVM 堆 `-Xmx256m`（可再省 ~50M）、`vm.swappiness=10`。

### D. 磁盘告警阈值：从「本机专项 20%/10%」回退为默认

- 演进：09-27 为本机加了 `project.disk` 专项（`JenkinsHostDiskLow` <20% / `JenkinsHostDiskCritical` <10%，比其他节点默认 5%/3% 更早预警）。
- 09-28 用户要求「**本节点和其他节点改成一样**」→ 撤掉本机特殊规则，全部节点回归 KPS 默认阈值。
- 执行：
  - 先误判为「其他节点向本机看齐（全 20%/10%）」，改成全节点 `NodeDiskAvailableLow/Critical`（20%/10%）——**方向反了**。
  - 用户确认是 **A**：「本节点向下对齐其他节点」→ **回滚**。删除 `project.disk` 组的自定义规则，全节点回到默认：
    - `NodeFilesystemAlmostOutOfSpace`：可用 <5% warning / <3% critical（for 30m）
    - `NodeFilesystemSpaceFillingUp`：可用 <15% 且预测 24h 满 → warning；<10% 且预测 4h 满 → critical（for 1h）
  - 保留 `project.jenkins`（构建失败/健康度/端点）与 `project.node`（内存 300M/150M）不动。
- **备份**：杭州 `/root/kps-project-alerts-backup-20260928-174424.yaml`（含原本机 20%/10% 规则，要恢复可用）。
- 备查各节点实时：杭州 54.4% / 上海 74.5% / home-node 19.7% / jenkins-host 70.3%（新默认下仅 <5% 才报）。

### E. 告警阈值调整小结（2026-09-28 全天）

> 这一天动了三处阈值/规则，统一收在这里备查。原则：**kps 默认阈值是按「标准规格节点」设计的，对 2 核小机、混跑节点偏敏感 → 用「新增放宽版 `project.*` 组 + 移除默认过于敏感的两条」代替全局改默认。**

#### E.1 节点饱和类：默认两条 → 换成 `project.saturation` 放宽版

| 项 | 原（kps 默认，node-exporter 组） | 现（`project.*` → `project.saturation`） |
|---|---|---|
| 系统负载 | `NodeSystemSaturation`：`load/core > 2` for 10m | `NodeSystemSaturation`：**`load/core > 4`** for 15m warning |
| 磁盘 IO | `NodeDiskIOSaturation`：`rate(node_disk_io_time_weighted…[5m]) > 10` for 30m | `NodeDiskIOSaturation`：**`> 20`** for 30m warning |

- **默认那两条已从 node-exporter 组彻底移除**（不再重复报警 ✅）。
- **理由**：杭州小机 2 核，`load/core > 4` ≈ 负载 8 才算真饱和；对「control-plane + APISIX + frp 混跑」的小规格机合理。
- **磁盘 IO `> 20`（原 10）**：给 `frps`/`openresty` 那些 **D 状态（不可中断 IO）** 进程留余量，避免常态基线就报。
- 移除后当时 `NodeDiskIOSaturation` 处于 pending（IO 值介于 10–20），实测当前 **5.28** → 后续会自己掉出 pending，不会 firing。

#### E.2 磁盘空间类（`project.disk`）：回退为默认

见上文 **D** 节 —— 本机专项 <20%/<10% 撤销，全节点回归 KPS 默认（`NodeFilesystemAlmostOutOfSpace` <5%/<3%；`NodeFilesystemSpaceFillingUp` <15%/<10% 且预测将满）。`project.disk` 组规则已删。

#### E.3 保留不动

- `project.jenkins`（构建失败 / 健康度 / 指标端点，`ci-lib-*` 试验 job 已排除）
- `project.node`（内存 300M / 150M）

#### E.4 关键坑：脚本生成的 YAML 删「数组项」必须用 `replace`，不能 `apply`

- **现象**：用 `kubectl apply -f` 更新 PrometheusRule，**想删掉的规则（数组元素）又被合回来了**，删不掉。
- **原因**：`kubectl apply` 是**三方合并**（last-applied / live / 新文件）。对于数组，它按合并键（这里是 `alert` 名）做「并集」，**文件里删掉的元素不会从 live 里移除** → 表现为「删了还在」。
- **解决**：改用 **`kubectl replace -f`** 强制整份覆盖 → 成功删除。
- **教训**：**凡是用脚本生成整份 YAML 去增删数组项（规则列表、容器列表、volumeMounts…），一律用 `kubectl replace`，不要 `apply`。**
  - `replace` 要求资源已存在（否则用 `apply`/`create`）；改前先 `kubectl get … -o yaml > backup`。
- 补充：PrometheusRule 改完要等 operator reconcile + config-reloader，`/api/v1/rules` 里看到 `health=ok` 才作数。

#### E.5 本次备份（杭州 `/root/`，随时可回滚）

| 文件 | 内容 |
|---|---|
| `kps-node-exporter-backup-20260928-235935.yaml` | node-exporter 组原始 **26 条规则**（含被移除的两条默认饱和规则） |
| `kps-project-alerts-backup-20260928-235626.yaml` | `project.*` 加 `saturation` 组**之前**的快照 |
| `kps-project-alerts-backup-20260928-174424.yaml` | 更早：磁盘阈值回退前（含本机 20%/10% 专项） |
| `kps-kube-apiserver-slos-backup-20260928-210210.yaml` | kube-apiserver SLO 放宽前 |

---

## 六、待办 / 延伸

- [ ] 转发器加**鉴权 header**（当前内网裸 HTTP，仅 WG 内可达；若要收紧加 token 校验）
- [x] 转发器**健壮性加固**（线程化 + socket 超时，修「单线程被半开连接堵死」，2026-09-28）
- [x] 补**核心告警规则**（Jenkins 构建 + 节点内存，2026-09-27）
- [ ] 同一出口**复用给 Jenkins 构建失败通知**
- [ ] 考虑告警分级路由（critical → 电话/更强提示；warning → 群）
- [ ] **Jenkins 构建看板**：已建 `Jenkins CI 效能看板`（uid `jenkins-ci-overview`，14 面板）
- [ ] 杭州小机减载/扩容预案（内存是问题 17 死因）

## 相关文档

- `04-k3s部署实战.md` — k3s 集群 + 监控栈部署（KPS 91.2.1）
- `notes/1-工作/02-凭证管理.md` — 企微 webhook 与转发器信息
- `knowledge/devops/kubernetes/05-监控详解.md` — Prometheus/Grafana/Loki 机制