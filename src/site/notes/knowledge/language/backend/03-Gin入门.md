---
{"dg-publish":true,"permalink":"/knowledge/language/backend/03-Gin入门/","title":"Gin 入门","tags":["go","gin","web","框架","restful","中间件"],"dg-note-properties":{"category":"后端 / Web 框架","title":"Gin 入门","created":"2026-11-01","tags":["go","gin","web","框架","restful","中间件"]}}
---


# Gin 入门

> Go 生态里最流行的 Web 框架。特点是**快**（基于 `httprouter`，路由零分配）和 **API 友好**（`c.JSON` 一行返回）。
> Python 侧对照见 [[knowledge/language/backend/01-Flask入门\|Flask 入门]]、[[knowledge/language/backend/02-Django入门\|Django 入门]]。

## 一、为什么是 Gin

| 框架 | 定位 | 特点 |
|:----|:----|:----|
| **标准库 `net/http`** | 官方 | 无依赖，但路由、参数绑定、JSON 都要手写 |
| **Gin** | 主流 Web 框架 | 路由快、中间件生态好、文档多、社区最大 |
| **Echo** | 同类 | API 设计更优雅，性能接近 |
| **Fiber** | 类 Express | 基于 fasthttp，快但不是标准 `net/http`，中间件不通用 |
| **Chi** | 轻量路由 | 纯 `net/http` 兼容，适合要自由的场景 |

> **选 Gin 的理由**：生态最大、`c.JSON()` 写起来最省事、几乎所有 Go 教程和公司项目都在用。

## 二、安装与最小示例

```bash
go mod init example.com/hello
go get -u github.com/gin-gonic/gin
```

`main.go`：

```go
package main

import "github.com/gin-gonic/gin"

func main() {
    r := gin.Default()          // 带 Logger + Recovery 中间件
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(200, gin.H{"message": "pong"})
    })
    r.Run(":8080")              // 默认监听 0.0.0.0:8080
}
```

```bash
go run main.go
curl http://127.0.0.1:8080/ping
# {"message":"pong"}
```

**`gin.Default()` vs `gin.New()`**

| | 内容 |
|:----|:----|
| `gin.Default()` | 内置 Logger（访问日志）+ Recovery（panic 恢复），**开发用** |
| `gin.New()` | 空白引擎，中间件全部自己加，**生产可控** |

## 三、路由

```go
r := gin.Default()

r.GET("/user", listUser)
r.POST("/user", createUser)
r.PUT("/user/:id", updateUser)
r.DELETE("/user/:id", deleteUser)
r.PATCH("/user/:id", patchUser)
r.Any("/any", anyHandler)           // 所有方法

// 路径参数
r.GET("/user/:id", func(c *gin.Context) {
    id := c.Param("id")
    c.String(200, "user %s", id)
})

// 通配（必须放最后）
r.GET("/static/*filepath", func(c *gin.Context) {
    c.String(200, c.Param("filepath"))
})

// 分组
api := r.Group("/api/v1")
{
    api.GET("/jobs", listJobs)
    api.POST("/jobs", createJob)
}

// 需要鉴权的分组
auth := r.Group("/admin", AuthMiddleware())
{
    auth.GET("/users", listUsers)
}
```

**注意**：Gin 的路由树**不允许** `:id` 与静态路径在同一层级冲突（如 `/user/:id` 和 `/user/new`），要么合并，要么换层级。

## 四、请求参数

### 4.1 查询参数

```go
func search(c *gin.Context) {
    kw := c.Query("kw")              // 不存在返回 ""
    page := c.DefaultQuery("page", "1")
    tags := c.QueryArray("tag")      // ?tag=a&tag=b
    c.JSON(200, gin.H{"kw": kw, "page": page, "tags": tags})
}
```

### 4.2 表单 / JSON 绑定（Gin 的核心便利）

用 **struct tag** 一次性绑定并校验：

```go
type CreateJobReq struct {
    Name     string `json:"name" binding:"required"`
    Repo     string `json:"repo" binding:"required,url"`
    Branch   string `json:"branch"`
    Timeout  int    `json:"timeout" binding:"gte=0,lte=3600"`
}

func createJob(c *gin.Context) {
    var req CreateJobReq
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"code": 400, "msg": err.Error()})
        return
    }
    c.JSON(200, gin.H{"code": 0, "data": req})
}
```

| 方法 | 用途 |
|:----|:----|
| `ShouldBindJSON` | 只绑 JSON，失败返回 error（**推荐**） |
| `BindJSON` | 失败自动 abort 并写 400 |
| `ShouldBind` | 按 Content-Type 自动选 |
| `ShouldBindQuery` | 绑定 query string |
| `ShouldBindUri` | 绑定路径参数 |

**常用 `binding` 校验 tag**

| tag | 含义 |
|:----|:----|
| `required` | 必填 |
| `email` / `url` / `uuid` | 格式 |
| `gte` / `lte` / `gt` / `lt` | 数值范围 |
| `min` / `max` / `len` | 长度 |
| `oneof=a b c` | 枚举 |
| `omitempty` | 空值跳过校验 |

### 4.3 文件上传

```go
func upload(c *gin.Context) {
    file, err := c.FormFile("file")
    if err != nil {
        c.JSON(400, gin.H{"msg": "no file"})
        return
    }
    // 安全：只用 filepath.Base，丢弃客户端给的目录
    dst := filepath.Join("uploads", filepath.Base(file.Filename))
    if err := c.SaveUploadedFile(file, dst); err != nil {
        c.JSON(500, gin.H{"msg": "save failed"})
        return
    }
    c.JSON(200, gin.H{"name": dst})
}
```

## 五、响应

```go
c.JSON(200, gin.H{"code": 0, "data": []string{}})     // 最常用
c.JSON(200, struct{ ID int }{1})                      // struct 也行
c.String(200, "ok")                                   // text/plain
c.XML(200, data)                                      // application/xml
c.File("a.pdf")                                       // 下载文件
c.Redirect(302, "/login")
c.AbortWithStatusJSON(401, gin.H{"msg": "unauthorized"})   // 中断并返回
c.Set("trace_id", id)                                 // 上下文传值
c.Get("trace_id")                                     // 中间件之间传值
```

**统一响应封装（实践写法）**

```go
type Resp struct {
    Code int         `json:"code"`
    Msg  string      `json:"msg"`
    Data interface{} `json:"data,omitempty"`
}

func OK(c *gin.Context, data interface{}) {
    c.JSON(200, Resp{Code: 0, Msg: "ok", Data: data})
}
func Fail(c *gin.Context, status, code int, msg string) {
    c.AbortWithStatusJSON(status, Resp{Code: code, Msg: msg})
}
```

## 六、中间件（Gin 的精髓）

中间件 = `func(c *gin.Context)`，**`c.Next()` 之前的代码在处理器前执行，之后的代码在处理器后执行**。

```go
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(401, gin.H{"msg": "missing token"})
            return                  // Abort 之后必须 return，否则继续往下走
        }
        c.Set("user", "admin")       // 传给后续 handler
        c.Next()                     // 继续执行后面的中间件/handler

        // c.Next() 之后 —— 用于统计耗时、记录响应
        c.Writer.Header().Set("X-Trace", "abc")
    }
}
```

**注册方式**

```go
r := gin.New()
r.Use(gin.Logger(), gin.Recovery())   // 全局
r.Use(AuthMiddleware())

api := r.Group("/api")
api.Use(RateLimit())                  // 分组级
```

**跨域中间件（手写版，省一个依赖）**

```go
func CORS() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("Access-Control-Allow-Origin", "*")
        c.Header("Access-Control-Allow-Methods", "GET,POST,PUT,DELETE,OPTIONS")
        c.Header("Access-Control-Allow-Headers", "Content-Type,Authorization")
        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(204)
            return
        }
        c.Next()
    }
}
```

## 七、项目结构（团队标准）

```
project/
├── main.go                 # 入口
├── go.mod
├── config/                 # 配置加载
├── router/                 # 路由注册
│   └── router.go
├── middleware/             # 中间件
├── handler/                # 控制器（接收请求）
├── service/                # 业务逻辑
├── model/                  # 数据结构 / 数据库模型
├── dao/                    # 数据访问
└── pkg/                    # 内部公共库
```

**分层价值**：`handler` 只管参数和响应，`service` 放业务，`dao` 管数据库 —— 测试和替换都方便。

## 八、生产化要点

```go
// 1. 关调试模式，去掉 gin 的 warn
gin.SetMode(gin.ReleaseMode)

// 2. 用 http.Server 控制超时（r.Run 没有超时控制）
srv := &http.Server{
    Addr:              ":8080",
    Handler:           r,
    ReadHeaderTimeout: 5 * time.Second,
    ReadTimeout:       30 * time.Second,
    WriteTimeout:      30 * time.Second,
    IdleTimeout:       60 * time.Second,
}
go srv.ListenAndServe()

// 3. 优雅退出
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
srv.Shutdown(ctx)
```

| 项 | 说明 |
|:----|:----|
| 健康检查 | 提供 `/healthz`，供 k8s liveness/readiness 探针 |
| 反向代理 | nginx / Ingress 终止 TLS |
| 日志 | 结构化 JSON 日志，带 trace id |
| 容器化 | 多阶段构建，最终镜像用 `scratch` 或 `distroless` |

```dockerfile
FROM golang:1.23-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/app .

FROM gcr.io/distroless/static-debian12
COPY --from=build /out/app /app
EXPOSE 8080
ENTRYPOINT ["/app"]
```

> `CGO_ENABLED=0` + `distroless` = 镜像通常 **< 20MB**，且几乎没有可攻击面。

## 九、常见坑

| 现象 | 原因 | 解法 |
|:----|:----|:----|
| 改了路由 404 | 静态路径与 `:param` 冲突 | 调整路由层级 |
| 绑定永远失败 | struct tag 写成 `form` 但发的是 JSON | JSON 用 `json:"..." binding:"..."` |
| `Abort` 后仍执行 | 忘了 `return` | `Abort` 后立即 `return` |
| prod 输出一堆 warning | 未 `SetMode(ReleaseMode)` | 设释放模式 |
| `r.Run()` 无超时 | 默认 `http.Server` 不设超时 | 自建 `http.Server` |
| 上传文件名带 `../` | 未过滤 | `filepath.Base()` |
| panic 导致进程挂 | 没用 Recovery | `gin.Recovery()` |
| 高并发内存涨 | 中间件里存大对象到 Context | Context 只传小值 |

## 十、一句话总结

> `gin.Default()` → 注册路由 → `ShouldBindJSON` 拿参数 → `c.JSON` 返回 → 中间件用 `c.Next()` 串起来 → 生产关 debug、设超时、开优雅退出。

## 相关文档

- [[knowledge/language/backend/01-Flask入门\|Flask 入门]]
- [[knowledge/language/backend/02-Django入门\|Django 入门]]
- Ubuntu 搭建 Golang 环境（见 `knowledge/devops/docker/` 相关文档）
