---
{"dg-publish":true,"permalink":"/knowledge/devops/docker/04-Compose实战与编排规范/","title":"Docker Compose 实战与编排规范","tags":["docker","compose","编排","容器","网络","数据卷","健康检查"],"dg-note-properties":{"category":"Docker / 容器 / DevOps","title":"Docker Compose 实战与编排规范","date":"2026-11-10","tags":["docker","compose","编排","容器","网络","数据卷","健康检查"]}}
---


# Docker Compose 实战与编排规范

> Compose 是单机多容器编排的事实标准，但很多项目把它当代替 `docker run` 的语法糖用——健康检查、启动顺序、网络隔离全都没配，最后靠 `restart: always` 兜底。
> 一句话主线：**Compose 的 `depends_on` 不管"能不能用"，只管"启没启动"；真正决定服务可用的，是 `healthcheck` 加 `condition: service_healthy`。**

---

## 一、先避开三个致命误区

### 误区 1：用 `docker-compose`（v1）

```bash
docker-compose up -d      # ❌ v1，Python 实现，已停止维护
docker compose up -d      # ✅ v2，Go 实现，插件形式
```

**v1.29.2 有一个破坏性 bug**：`KeyError: 'ContainerConfig'`——重新 `up` 时它会**把旧容器删掉但不建新的**，服务直接消失。

**判断当前用的是哪个**：

```bash
docker-compose version     # v1，输出带 "docker-compose version 1.x"
docker compose version     # v2，输出 "Docker Compose version v2.x"
```

**如果只有 v1**，装 v2 插件：

```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
docker compose version
```

---

### 误区 2：以为 `depends_on` 能保证顺序

```yaml
services:
  app:
    depends_on:
      - db        # ❌ 只保证 db 容器「启动」，不保证「能连」
```

`depends_on` 的默认语义是「先启动 db，再启动 app」，但 db 容器起来了 ≠ 数据库初始化完了。app 连过去照样 `connection refused`。

**正确写法**：

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy    # ✅ 等 db 健康检查通过
  db:
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s
```

**这是 Compose 最容易被忽略的一条。** 没有 `condition`，`depends_on` 基本等于没写。

---

### 误区 3：数据不放 volume

```yaml
volumes:
  - ./data:/var/lib/postgresql/data    # ⚠️ 能用，但有坑
```

bind mount 到宿主机目录的问题：

| 问题 | 说明 |
|:--|:--|
| 权限 | 容器内 uid 与宿主机 uid 不一致 → 写不进去 |
| 备份 | 手动拷贝，容易漏文件 |
| 迁移 | 路径写死，换机器要改 |
| SELinux | 需要 `:z` / `:Z` 标签 |

**推荐**：数据用 named volume，配置文件才用 bind mount。

```yaml
volumes:
  pgdata:                          # named volume，Docker 管理

services:
  db:
    volumes:
      - pgdata:/var/lib/postgresql/data     # 数据
      - ./conf/pg.conf:/etc/postgresql.conf:ro   # 配置（只读）
```

---

## 二、一份生产可用的模板

```yaml
name: myapp                      # 项目名，影响容器/网络/卷前缀

services:
  web:
    image: nginx:1.27-alpine
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:80"      # 关键：绑 127.0.0.1，不暴露到公网
    volumes:
      - ./conf/nginx.conf:/etc/nginx/nginx.conf:ro
      - static:/usr/share/nginx/html:ro
    depends_on:
      api:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost/healthz"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 5s
    networks: [frontend]
    logging:
      driver: json-file
      options: { max-size: "50m", max-file: "3" }
    deploy:
      resources:
        limits: { cpus: "1.0", memory: 256M }

  api:
    build:
      context: ./api
      dockerfile: Dockerfile
      args:
        GO_VERSION: "1.23"
    image: myapp-api:${TAG:-latest}   # 用环境变量打版本
    restart: unless-stopped
    environment:
      DATABASE_URL: postgres://app:${DB_PASS:?DB_PASS required}@db:5432/app
      TZ: Asia/Shanghai
    env_file:
      - .env.local                     # 本地覆盖，不进仓库
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/healthz"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 20s
    networks: [frontend, backend]
    logging:
      driver: json-file
      options: { max-size: "50m", max-file: "3" }

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASS:?DB_PASS required}
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 15s
    networks: [backend]              # 只在 backend，web 访问不到

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes", "--maxmemory", "256mb", "--maxmemory-policy", "allkeys-lru"]
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks: [backend]

volumes:
  pgdata:
  redisdata:
  static:

networks:
  frontend:
  backend:
    internal: true                   # 关键：backend 无外网出口
```

### 几处设计说明

| 写法 | 为什么 |
|:--|:--|
| `ports: "127.0.0.1:8080:80"` | 只绑本地，公网访问不到，靠上游反代 |
| `networks: backend + internal: true` | db/redis 不能在 backend 里有外网出口，防横移 |
| `${DB_PASS:?DB_PASS required}` | 变量缺失直接报错，而不是用空密码启动 |
| `logging.options.max-size` | 不配这个，日志能把磁盘撑爆 |
| `deploy.resources.limits` | 单机 v2 也生效，防单个容器吃光内存 |
| `healthcheck.start_period` | 启动期不计入失败次数，避免误判 |

---

## 三、生命周期命令

```bash
# 启动（后台）
docker compose up -d

# 改了镜像要重建
docker compose up -d --build

# 只看配置（不启动），排查变量替换
docker compose config

# 状态
docker compose ps

# 日志
docker compose logs -f --tail=100 api

# 单服务重启
docker compose restart api

# 扩缩容（无端口冲突的前提下）
docker compose up -d --scale api=3

# 停止（保留容器）
docker compose stop

# 停止并删除容器/网络（保留卷）
docker compose down

# 连卷一起删（危险）
docker compose down -v

# 删除孤儿容器（改了 service 名之后用）
docker compose up -d --remove-orphans
```

### 最常用的三条

```bash
docker compose config          # 配置对不对
docker compose up -d --build   # 改了代码/镜像
docker compose logs -f <服务>  # 出问题看日志
```

---

## 四、变更生效规则（最容易踩的坑）

**核心原则：bind mount 的文件改了，不一定生效。**

| 变的是什么 | 需要做什么 |
|:--|:--|
| bind mount 的配置文件（如 nginx.conf）| **`docker compose restart`**（容器内进程要重读配置）|
| 代码（如果代码是 COPY 进镜像的）| `docker compose up -d --build` |
| 代码（如果代码是 bind mount 的）| 看语言：Python 需重启进程，Node 热重载可能自动生效 |
| `compose.yaml` 本身 | `docker compose up -d`（Compose 会重建有变化的服务）|
| `.env` 里的变量 | `docker compose up -d`（需要重建容器才注入新环境变量）|
| 镜像 tag 指向新镜像 | `docker compose up -d --pull always` |

> **反例**：改了 `nginx.conf`，只跑 `docker compose up -d` → 容器没变，Compose 认为配置没改 → **配置不生效**。
> 因为 Compose 只检查 `compose.yaml` 的差异，**不检查 bind mount 内容的差异**。

---

## 五、多环境管理

### 5.1 用 override 文件叠加

```
compose.yaml                 # 基础
compose.override.yaml        # 本地开发（自动加载，不进仓库）
compose.prod.yaml            # 生产
```

```bash
# 开发（自动带上 override）
docker compose up -d

# 生产（显式叠加）
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

`compose.prod.yaml` 只写差异：

```yaml
services:
  web:
    ports:
      - "80:80"           # 生产绑 0.0.0.0
  api:
    restart: always
    deploy:
      resources:
        limits: { cpus: "2.0", memory: 1G }
```

> **注意**：`ports` 是**列表，会整体覆盖**不是追加。基础文件写 `127.0.0.1:8080:80`，override 写 `80:80`，结果是 `80:80`。

### 5.2 用 .env 做变量注入

`.env`（Compose 自动加载，用于变量替换）：

```ini
TAG=1.2.3
DB_PASS=changeme
```

`compose.yaml` 里引用 `${TAG}`，`docker compose up` 时替换。

**三层变量优先级**（从高到低）：

```
1. 命令行：docker compose run -e FOO=bar
2. shell 环境变量：export FOO=bar
3. .env 文件
4. compose.yaml 里的默认值 ${FOO:-default}
```

---

## 六、调试技巧

### 6.1 看容器为什么退出

```bash
docker compose ps -a           # 看 ExitCode
docker compose logs --tail=200 <服务>
```

### 6.2 进容器但不走 entrypoint

```bash
docker compose run --rm --entrypoint sh api
```

### 6.3 检查健康检查状态

```bash
docker inspect --format '{{json .State.Health}}' <容器名> | jq
```

### 6.4 检查网络连通性

```bash
# 从 web 容器 ping db
docker compose exec web sh -c "getent hosts db"

# 从宿主机看网络
docker network inspect myapp_backend
```

### 6.5 配置有问题怎么看

```bash
docker compose config          # 渲染后的完整配置（变量已替换）
docker compose config --quiet  # 只校验语法，不输出
```

> `docker compose config` 是排查"为什么变量没生效"的最快手段——它显示的是**最终渲染结果**。

---

## 七、常见报错

| 报错 | 原因 | 解法 |
|:--|:--|:--|
| `KeyError: 'ContainerConfig'` | v1 的 bug | 换 v2 |
| `port is already allocated` | 端口被占 | `ss -lntp \| grep <端口>` 找占用者 |
| `no such service: xxx` | service 名拼错或文件没加载 | `docker compose config --services` |
| `pull access denied` | 镜像私有或不存在 | `docker login` |
| `network xxx not found` | 网络被 `down` 删了 | 重新 `up` |
| `container name already in use` | 残留容器 | `docker compose down` 或 `docker rm -f` |
| 变量是空的 | `.env` 没加载 / 拼写错 | `docker compose config` 核对 |
| `permission denied` 写 bind mount | uid 不匹配 | 用 named volume，或 `user:` 指定 uid |
| 健康检查一直 unhealthy | `start_period` 太短 | 加长 `start_period` |

---

## 八、v1 迁移到 v2 注意点

| 项 | v1 | v2 |
|:--|:--|:--|
| 命令 | `docker-compose` | `docker compose` |
| 文件默认名 | `docker-compose.yml` | `compose.yaml`（yml 仍支持）|
| `version:` 字段 | 必须有 | **已废弃**，写了会报警告 |
| `deploy` 段 | 只在 swarm 生效 | **单机也生效** |
| 环境变量文件 | `--env-file` 默认 `.env` | 同 |

**迁移检查清单**：

```
□ 删掉 version: 字段
□ docker-compose.* → docker compose（空格）
□ 确认 deploy.resources 的行为（单机现在生效了，可能限制过严）
□ compose.override.yaml 命名（v1 的 docker-compose.override.yml 仍可）
```

---

## 九、一句话总结

> **Compose 写对了能省一个 K8s，写错了就是一个更大的 `docker run`。**
> 三条底线：**`depends_on` 必须配 `condition: service_healthy`**、**数据必须用 named volume**、**每个服务必须有 `healthcheck` 和日志限额**。这三条不做，编排就不成立，只是把命令拼在一个文件里。
