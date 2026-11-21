---
{"dg-publish":true,"permalink":"/knowledge/course/csapp-labs/08-Proxy-Lab/","title":"CSAPP Lab8 · Proxy Lab","tags":["CSAPP","LAB","实验","网络","并发","缓存代理"],"dg-note-properties":{"title":"CSAPP Lab8 · Proxy Lab","tags":["CSAPP","LAB","实验","网络","并发","缓存代理"]}}
---


# Proxy Lab

写一个带缓存的并发 HTTP 代理。三部分递进，官方 `driver.sh` 自动评分，总分 70。

| 部分 | 内容 | 分值 |
| --- | --- | ---: |
| Basic | 顺序代理：解析请求、转发、回传 | 40 |
| Concurrency | 每个连接一个线程，处理并发 | 15 |
| Cache | LRU 缓存已访问对象 | 15 |

**实测结果：70/70 满分。**

```text
basicScore: 40/40
concurrencyScore: 15/15
cacheScore: 15/15
totalScore: 70/70
```

---

## 环境准备

**问题 1：`nop-server.py` 的 shebang 指向 `/usr/bin/python`**

现代系统通常只有 `python3`，并发测试会卡住：

```text
Starting the blocking NOP server on port 30126
Timeout waiting for the server to grab the port reserved for it
```

改第一行：

```bash
sed -i '1s|.*|#!/usr/bin/env python3|' nop-server.py
```

**问题 2：`P()` / `V()` 签名是 `sem_t*`**

`csapp.h` 里：

```c
void P(sem_t *sem);
void V(sem_t *sem);
```

如果你用 `pthread_mutex_t` 做锁，别用 `P`/`V`，直接调 `pthread_mutex_lock` / `pthread_mutex_unlock`，否则编译警告、运行时行为也不对。

**编译**

```bash
make                    # 编译 proxy
cd tiny && make         # 编测试服务器
```

---

## HTTP 代理要做什么

```text
浏览器 ──请求──> 代理 ──转发──> 目标服务器
        <──响应──      <──响应──
```

代理需要：
1. 读客户端请求行 `GET http://host:port/path HTTP/1.0`
2. 从 URI 里拆出 host / port / path
3. 连到目标服务器，发一个精简的请求
4. 读响应，原样回传客户端
5. 顺手把响应存进缓存（第二次访问直接返回）

### 必须注意的 HTTP 细节

| 项 | 说明 |
| --- | --- |
| 只支持 GET | 其他方法返回 `501` |
| 用 HTTP/1.0 | 避免 chunked 编码、持久连接等复杂情况 |
| 发 `Host:` 头 | HTTP/1.1 强制要求，但 1.0 也要带，很多服务器看 |
| 发 `Connection: close` | 否则服务器可能保持连接，你读不到 EOF |
| 发 `Proxy-Connection: close` | 同上 |
| 跳过客户端的头 | `Host`、`User-Agent`、`Connection` 等自己重写，其余可丢 |

---

## 一、解析 URI

输入形如：

```text
http://localhost:8080/home.html
http://www.example.com/index.html
```

要拆成 `host` / `port` / `path`：

```c
static void parse_uri(const char *uri, char *host, char *port, char *path)
{
    const char *p = uri;

    if (!strncmp(p, "http://", 7)) p += 7;

    const char *colon = strchr(p, ':');
    const char *slash = strchr(p, '/');

    if (colon && (!slash || colon < slash)) {
        /* 有端口：host:port/path */
        size_t hlen = colon - p;
        memcpy(host, p, hlen);
        host[hlen] = '\0';
        if (slash) {
            size_t plen = strcspn(colon + 1, "/");
            memcpy(port, colon + 1, plen);
            port[plen] = '\0';
            strcpy(path, slash);
        } else {
            strcpy(port, colon + 1);
            strcpy(path, "/");
        }
    } else {
        /* 没端口，默认 80 */
        if (slash) {
            size_t hlen = slash - p;
            memcpy(host, p, hlen);
            host[hlen] = '\0';
            strcpy(path, slash);
        } else {
            strcpy(host, p);
            strcpy(path, "/");
        }
        strcpy(port, "80");
    }
}
```

**关键判断**：`colon && (!slash || colon < slash)` —— 冒号必须在斜杠之前才算端口分隔符。否则 `http://host/path:foo` 会把 `path:foo` 误当端口。

---

## 二、Core：处理单个连接

```c
static void doit(int fd)
{
    char buf[MAXLINE];
    char method[MAXLINE], uri[MAXLINE], version[MAXLINE];
    char host[MAXLINE], port[16], path[MAXLINE];
    char request[MAXBUF], url[MAXLINE];
    rio_t rio_client, rio_server;
    int serverfd;
    char *cachebuf = NULL;
    size_t cachelen = 0;
    cache_block *hit;
    size_t n;

    /* 1. 读请求行 */
    Rio_readinitb(&rio_client, fd);
    if (Rio_readlineb(&rio_client, buf, MAXLINE) <= 0)
        return;
    if (sscanf(buf, "%s %s %s", method, uri, version) != 3)
        return;

    if (strcasecmp(method, "GET")) {
        Rio_writen(fd, "HTTP/1.0 501 Not Implemented\r\n\r\n", 35);
        return;
    }

    parse_uri(uri, host, port, path);
    snprintf(url, sizeof url, "http://%s:%s%s", host, port, path);

    /* 2. 查缓存 */
    pthread_mutex_lock(&cache_lock);
    hit = cache_find(url);
    if (hit) {
        cachebuf = Malloc(hit->size);
        memcpy(cachebuf, hit->data, hit->size);
        cachelen = hit->size;
    }
    pthread_mutex_unlock(&cache_lock);

    if (cachebuf) {
        Rio_writen(fd, cachebuf, cachelen);
        Free(cachebuf);
        return;
    }

    /* 3. 连服务器 */
    serverfd = Open_clientfd(host, port);
    if (serverfd < 0) {
        Rio_writen(fd, "HTTP/1.0 502 Bad Gateway\r\n\r\n", 29);
        return;
    }

    /* 4. 构造请求 */
    snprintf(request, sizeof request,
             "GET %s HTTP/1.0\r\n"
             "Host: %s:%s\r\n"
             "%s"
             "Connection: close\r\n"
             "Proxy-Connection: close\r\n\r\n",
             path, host, port, user_agent_hdr);

    /* 丢掉客户端剩下的头 */
    while (Rio_readlineb(&rio_client, buf, MAXLINE) > 0) {
        if (!strcmp(buf, "\r\n")) break;
    }

    Rio_writen(serverfd, request, strlen(request));

    /* 5. 读响应，边转发边缓存 */
    Rio_readinitb(&rio_server, serverfd);
    {
        char *acc = Malloc(MAX_OBJECT_SIZE);
        size_t acc_len = 0;
        int too_big = 0;

        while ((n = Rio_readnb(&rio_server, buf, MAXLINE)) > 0) {
            Rio_writen(fd, buf, n);
            if (!too_big) {
                if (acc_len + n <= MAX_OBJECT_SIZE) {
                    memcpy(acc + acc_len, buf, n);
                    acc_len += n;
                } else {
                    too_big = 1;      /* 超过单对象上限，放弃缓存 */
                }
            }
        }
        if (!too_big && acc_len > 0) {
            pthread_mutex_lock(&cache_lock);
            cache_add(url, acc, acc_len);
            pthread_mutex_unlock(&cache_lock);
        }
        Free(acc);
    }

    Close(serverfd);
}
```

**四个易错点**

**1. 必须丢掉客户端剩余的头**

`Rio_readlineb` 只读了请求行，后面的 `Host:`、`User-Agent:` 等还在缓冲区里。如果不读掉，这些残留会被当成服务器的响应内容——而它们是发往服务器的，发错方向了。

循环读到空行 `\r\n` 结束即可。

**2. `Rio_readnb` 而不是 `Rio_readlineb`**

响应可能是二进制（图片），按行读会破坏数据。必须按字节读。

**3. 边转发边缓存**

不能等服务读完再转发——那样客户端要等整个响应缓存完才看到第一个字节。边读边写，同时累积到缓冲区。

**4. 超过上限就放弃缓存**

`MAX_OBJECT_SIZE` 是 100KB。如果响应更大，标记 `too_big`，不再累积，但**继续转发**（不能因为不缓存就不给客户端）。

---

## 三、并发：每连接一线程

```c
static void *thread(void *vargp)
{
    int fd = *((int *)vargp);
    Pthread_detach(Pthread_self());
    Free(vargp);
    doit(fd);
    Close(fd);
    return NULL;
}

int main(int argc, char **argv)
{
    int listenfd, *connfd;
    socklen_t clientlen;
    struct sockaddr_storage clientaddr;
    pthread_t tid;

    if (argc != 2) {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        exit(1);
    }

    listenfd = Open_listenfd(argv[1]);
    while (1) {
        clientlen = sizeof clientaddr;
        connfd = Malloc(sizeof(int));       /* 堆上分配，避免竞态 */
        *connfd = Accept(listenfd, (SA) &clientaddr, &clientlen);
        Pthread_create(&tid, NULL, thread, connfd);
    }
}
```

**为什么 `connfd` 必须 `Malloc`**

如果写成：

```c
int connfd;
while (1) {
    connfd = Accept(...);
    Pthread_create(&tid, NULL, thread, &connfd);   /* 错！ */
}
```

所有线程拿到的是**同一个栈变量的地址**，主线程一循环就改掉它，线程读到的 fd 可能是别人的。这是并发编程最经典的坑之一。

**必须 `Pthread_detach`**

否则线程结束后资源不回收，长时间运行会耗尽。`detach` 让线程结束时自动清理。

**并发测试的原理**

`driver.sh` 起一个 `nop-server.py`——它接受连接后永不响应。如果代理是顺序的，这个请求会把整个代理卡住，后续正常请求全部超时。只有每连接一线程（或多进程）才能同时处理。

---

## 四、缓存：LRU 双向链表

```c
typedef struct cache_block {
    char *url;
    char *data;
    size_t size;
    struct cache_block *prev, *next;    /* LRU 链表 */
} cache_block;

static cache_block *cache_head;         /* 最近使用 */
static cache_block *cache_tail;         /* 最久未用 */
static size_t cache_total;
static pthread_mutex_t cache_lock = PTHREAD_MUTEX_INITIALIZER;
```

**链表方向约定**：头 = 最新，尾 = 最旧。淘汰时摘尾。

```c
static void unlink_block(cache_block *b)
{
    if (b->prev) b->prev->next = b->next;
    else cache_head = b->next;
    if (b->next) b->next->prev = b->prev;
    else cache_tail = b->prev;
    b->prev = b->next = NULL;
}

static void link_front(cache_block *b)
{
    b->prev = NULL;
    b->next = cache_head;
    if (cache_head) cache_head->prev = b;
    cache_head = b;
    if (!cache_tail) cache_tail = b;
}

static void cache_evict_one(void)
{
    cache_block *b = cache_tail;
    if (!b) return;
    unlink_block(b);
    cache_total -= b->size;
    Free(b->url);
    Free(b->data);
    Free(b);
}
```

**查找时顺便提升优先级**

```c
static cache_block *cache_find(const char *url)
{
    cache_block *b;
    for (b = cache_head; b; b = b->next) {
        if (!strcmp(b->url, url)) {
            unlink_block(b);
            link_front(b);              /* 命中即变最近使用 */
            return b;
        }
    }
    return NULL;
}
```

这就是 LRU 的核心：**每次命中都把它移到链表头**，尾部自然是最久未用的。

**插入**

```c
static void cache_add(const char *url, const char *data, size_t size)
{
    cache_block *b;

    if (size > MAX_OBJECT_SIZE) return;

    /* 空间不够就淘汰尾部的 */
    while (cache_total + size > MAX_CACHE_SIZE && cache_tail)
        cache_evict_one();

    b = Malloc(sizeof(cache_block));
    b->url = strdup(url);
    b->data = Malloc(size);
    memcpy(b->data, data, size);
    b->size = size;
    link_front(b);
    cache_total += size;
}
```

**锁的粒度**

```c
/* 查缓存：锁内只拷贝数据，锁外发送 */
pthread_mutex_lock(&cache_lock);
hit = cache_find(url);
if (hit) {
    cachebuf = Malloc(hit->size);
    memcpy(cachebuf, hit->data, hit->size);
    cachelen = hit->size;
}
pthread_mutex_unlock(&cache_lock);

if (cachebuf) {
    Rio_writen(fd, cachebuf, cachelen);   /* 锁外发送，不阻塞其他线程 */
    ...
}
```

**不能在锁内 `Rio_writen`** —— 网络写可能阻塞几百毫秒，会让所有其他线程排队等锁，并发直接退化成串行。

**为什么不在链表中直接返回指针**

如果在锁内返回 `hit` 指针，解锁后另一个线程可能淘汰这个块，指针就悬空了。所以锁内必须**把数据拷贝出来**。

---

## 五、评分测试

```bash
make
cd tiny && make && cd ..
./driver.sh
```

**Basic（40 分）**：拉 5 个文件，分别经代理和直连，比对内容

```text
1: home.html
   Success: Files are identical.
2: csapp.c
   Success: Files are identical.
3: tiny.c
   Success: Files are identical.
4: godzilla.jpg
   Success: Files are identical.
5: tiny
   Success: Files are identical.
basicScore: 40/40
```

含文本、二进制、可执行文件，都能正确转发。

**Concurrency（15 分）**：先让一个请求撞上永不响应的 nop-server，再看正常请求能否通过

```text
Starting the blocking NOP server on port 25384
Trying to fetch a file from the blocking nop-server
Fetching ./tiny/home.html into ./.noproxy directly from Tiny
Fetching ./tiny/home.html into ./.proxy using the proxy
Checking whether the proxy fetch succeeded
Success: Was able to fetch tiny/home.html from the proxy.
concurrencyScore: 15/15
```

顺序代理会卡在第一个请求上，这里证明多线程生效了。

**Cache（15 分）**：拉几个文件 → **杀掉 tiny 服务器** → 再拉一次，能拉到说明来自缓存

```text
Fetching ./tiny/tiny.c into ./.proxy using the proxy
Fetching ./tiny/home.html into ./.proxy using the proxy
Fetching ./tiny/csapp.c into ./.proxy using the proxy
Killing tiny
Fetching a cached copy of ./tiny/home.html into ./.noproxy
Success: Was able to fetch tiny/home.html from the cache.
cacheScore: 15/15
```

**总成绩**

```text
totalScore: 70/70
```

---

## 六、手动测试

**起服务器和代理**

```bash
cd tiny && ./tiny 8000 &
cd .. && ./proxy 8080 &
```

**用 curl 走代理**

```bash
curl -x http://localhost:8080 http://localhost:8000/home.html
```

**验证缓存**

```bash
curl -x http://localhost:8080 http://localhost:8000/home.html  # 第一次，miss
kill %1                                                        # 杀掉 tiny
curl -x http://localhost:8080 http://localhost:8000/home.html  # 第二次，应命中缓存
```

第二次如果还能返回内容，说明缓存生效。

**看代理的请求**

启动前加个环境变量输出 debug 信息，或者用 `tcpdump`/`wireshark` 抓包看实际发出的 HTTP 头。

---

## 七、常见问题

**`501 Not Implemented`**

方法不是 GET。注意 `strcasecmp` 大小写不敏感，但要看 `method` 是否真的解析出来了——`sscanf` 返回值不是 3 说明请求行格式有问题。

**`502 Bad Gateway`**

`Open_clientfd` 失败。检查 host/port 解析是否正确，特别是 `parse_uri` 里冒号和斜杠的顺序判断。

**响应内容乱码 / 图片打不开**

用 `Rio_readlineb` 读响应了。必须用 `Rio_readnb` 按字节读。

**并发测试超时**

代理是顺序的，或者 `connfd` 用了栈变量地址（所有线程共享同一个 fd）。

**缓存测试失败**

- 缓存键用了完整 URL 还是只用 path？不同 host 相同 path 会串。
- 存的是完整响应（含 HTTP 头）还是只有 body？必须存**完整响应**，包括状态行和头，否则客户端解析失败。
- 淘汰策略写错，刚存的就被踢了。

**`driver.sh` 卡在 nop-server**

`nop-server.py` 的 shebang 是 `/usr/bin/python`，系统没有。改成 `python3`。

**编译警告 `passing argument 1 of 'P' from incompatible pointer type`**

`P`/`V` 接收 `sem_t *`，你传了 `pthread_mutex_t *`。改用 pthread 原生锁函数。

---

## 八、还能改进什么

**用哈希表加速查找**

当前 `cache_find` 遍历整个链表，O(n)。20 个缓存对象没问题，但如果改成细粒度缓存或大容量，应该加哈希表，把查找降到 O(1)。

**支持 HTTPS CONNECT**

现在的实现只处理 `http://`。真正的代理要处理 `CONNECT host:443` 建立隧道，那是字节流双向转发，不解析 HTTP。

**更精细的缓存策略**

当前不检查响应头里的 `Cache-Control`、`Expires`。严格实现应该尊重这些，`no-store` 就不缓存，`max-age` 到期就失效。

**连接池**

每次请求都新建到服务器的连接。可以用连接池复用，减少 TCP 握手开销。

---

## 附录：环境与踩坑

### 问题 1：`nop-server.py` 的 shebang 指向不存在的 python

并发测试会卡在这里：

```bash
./driver.sh
```
```text
*** Concurrency ***
Starting tiny on port 20335
Starting proxy on port 8377
Starting the blocking NOP server on port 30126
Timeout waiting for the server to grab the port reserved for it
```

看起来像代理的问题，其实是 `nop-server.py` 根本没起来。看第一行：

```bash
head -1 nop-server.py
```
```text
#!/usr/bin/python
```

现代系统通常只有 `python3`：

```bash
which python python3
```
```text
/usr/bin/python3
```

`/usr/bin/python` 不存在 → 脚本无法执行 → 端口一直没人监听 → driver 等超时。

**修复**：

```bash
sed -i '1s|.*|#!/usr/bin/env python3|' nop-server.py
head -2 nop-server.py
```
```text
#!/usr/bin/env python3
```

确认 Python 3 语法兼容：

```bash
python3 -c "import ast; ast.parse(open('nop-server.py').read())" && echo OK
```

改完后 concurrency 从超时变成 15/15。

**排查思路**：driver 报"等端口超时"时，先单独手动起那个服务看能不能起来，别一头扎进代理代码里。

### 问题 2：`P()` / `V()` 接收 `sem_t*` 不是 mutex

`csapp.h` 的声明：

```c
void P(sem_t *sem);
void V(sem_t *sem);
```

如果用 `pthread_mutex_t` 做锁却调 `P`/`V`：

```c
static pthread_mutex_t cache_lock = PTHREAD_MUTEX_INITIALIZER;
P(&cache_lock);       /* 编译警告 */
```

```text
proxy.c:171:7: warning: passing argument 1 of 'P' from incompatible pointer type
```

`sem_wait` 和 `pthread_mutex_lock` 的底层实现不同，传错类型行为不可预测。

**正确做法**：直接用 pthread 原生函数。

```c
pthread_mutex_lock(&cache_lock);
/* 临界区 */
pthread_mutex_unlock(&cache_lock);
```

或者用信号量：

```c
static sem_t cache_sem;
Sem_init(&cache_sem, 0, 1);
P(&cache_sem);
V(&cache_sem);
```

两种都行，别混用。

### 问题 3：`tiny` 必须先编译

`driver.sh` 检查 `./tiny/tiny` 是否存在且可执行：

```text
if [ ! -x ./tiny/tiny ]; then
    echo "Error: ./tiny/tiny not found or not executable."
```

```bash
cd tiny && make && cd ..
ls -la tiny/tiny
```
```text
-rwxr-xr-x 1 root root 36240 ... tiny/tiny
```

`tiny` 本身编译会有一堆 `-Wformat-overflow` 警告（`tiny.c` 和 `adder.c` 里的 `sprintf` 用法），但**不影响生成可执行文件**，忽略即可。

### 问题 4：并发版 `connfd` 用栈变量

这是并发编程最经典的错误：

```c
/* 错误写法 */
int connfd;
while (1) {
    connfd = Accept(listenfd, ...);
    Pthread_create(&tid, NULL, thread, &connfd);   /* 传栈地址 */
}
```

所有线程拿到的是**同一个栈变量的地址**。主线程下一轮循环立刻改掉它，线程读到的可能是别人的 fd。

现象很诡异：有时正常，有时响应串到别的连接，有时直接崩溃，完全取决于线程调度时序。

**正确写法**：堆上分配。

```c
while (1) {
    connfd = Malloc(sizeof(int));
    *connfd = Accept(listenfd, ...);
    Pthread_create(&tid, NULL, thread, connfd);
}
```

线程里负责释放：

```c
static void *thread(void *vargp)
{
    int fd = *((int *)vargp);
    Pthread_detach(Pthread_self());
    Free(vargp);              /* 线程自己释放 */
    doit(fd);
    Close(fd);
    return NULL;
}
```

### 必须 `Pthread_detach`

```c
Pthread_detach(Pthread_self());
```

不加的话线程结束后资源不回收（内核保留退出状态等你 `join`）。长时间运行会耗尽线程资源。

因为主线程从不 `join`，所以必须 `detach`。

### 响应读要用 `Rio_readnb` 不是 `Rio_readlineb`

```c
/* 错误：按行读会破坏二进制 */
while (Rio_readlineb(&rio_server, buf, MAXLINE) > 0)
    Rio_writen(fd, buf, strlen(buf));

/* 正确：按字节读 */
while ((n = Rio_readnb(&rio_server, buf, MAXLINE)) > 0)
    Rio_writen(fd, buf, n);
```

图片、可执行文件里随时可能出现 `\n` 字节，按行读会把数据切碎。而且 `strlen` 遇到 `\0` 会截断。

基础测试里的 `godzilla.jpg` 和 `tiny`（可执行文件）就是专门测这个的 —— 用按行读会失败。

### 必须丢掉客户端剩余的头

```c
while (Rio_readlineb(&rio_client, buf, MAXLINE) > 0) {
    if (!strcmp(buf, "\r\n")) break;
}
```

`Rio_readlineb` 只读了请求行 `GET ... HTTP/1.0\r\n`，后面的 `Host:`、`User-Agent:` 等还在缓冲区里。

如果直接去连服务器发请求，这些残留还留在 client 的 rio 缓冲区里 —— 不影响发包，但下次读这个 fd 会读到它们。而且更重要的是：**这些头没有转发给服务器**（我们重建了请求），所以必须显式读掉，保持流同步。

不读掉的话，某些情况下 client 端会认为连接没处理完。

### `Connection: close` 和 `Proxy-Connection: close` 都要发

```c
snprintf(request, sizeof request,
         "GET %s HTTP/1.0\r\n"
         "Host: %s:%s\r\n"
         "%s"
         "Connection: close\r\n"
         "Proxy-Connection: close\r\n\r\n",
         path, host, port, user_agent_hdr);
```

**为什么必须 `Connection: close`**：HTTP/1.0 默认短连接，但现代服务器看到 `Host` 头可能会升级行为。明确要求关闭，服务器回完响应就关连接，你才能读到 EOF 结束循环。

不加的话 `Rio_readnb` 可能永远阻塞（服务器等下一个请求，你等更多数据）。

`Proxy-Connection` 是给中间代理看的，老规范里的头，同样写上更保险。

### 缓存必须存完整响应，不只是 body

```c
/* 缓存 acc 里累积的是完整响应：状态行 + 头 + body */
while ((n = Rio_readnb(&rio_server, buf, MAXLINE)) > 0) {
    Rio_writen(fd, buf, n);
    if (!too_big && acc_len + n <= MAX_OBJECT_SIZE) {
        memcpy(acc + acc_len, buf, n);
        acc_len += n;
    }
}
```

命中时直接原样返回：

```c
Rio_writen(fd, cachebuf, cachelen);
```

如果只存 body，命中时要自己重新拼 `HTTP/1.0 200 OK\r\nContent-Length: ...` 等头，容易出错（长度算错、缺 `Content-Type`）。

**存完整响应最简单也最正确。**

### 锁内不能做网络写

```c
/* 错误：锁内写网络 */
pthread_mutex_lock(&cache_lock);
hit = cache_find(url);
if (hit) Rio_writen(fd, hit->data, hit->size);   /* 可能阻塞几百毫秒 */
pthread_mutex_unlock(&cache_lock);
```

`Rio_writen` 遇到客户端慢消费会阻塞。所有其他线程排队等锁，并发直接退化成串行 —— 并发测试就过不了了。

**正确做法**：锁内只拷贝数据，锁外发送。

```c
pthread_mutex_lock(&cache_lock);
hit = cache_find(url);
if (hit) {
    cachebuf = Malloc(hit->size);
    memcpy(cachebuf, hit->data, hit->size);   /* 拷贝出来 */
    cachelen = hit->size;
}
pthread_mutex_unlock(&cache_lock);

if (cachebuf) {
    Rio_writen(fd, cachebuf, cachelen);       /* 锁外发 */
    Free(cachebuf);
}
```

### 为什么不能在锁内返回链表节点指针

```c
/* 错误 */
pthread_mutex_lock(&cache_lock);
cache_block *b = cache_find(url);
pthread_mutex_unlock(&cache_lock);
Rio_writen(fd, b->data, b->size);   /* b 可能已被别的线程淘汰释放 */
```

解锁后另一个线程可能因为空间不足把这个块踢掉并 `Free`，`b` 就是悬空指针。

**必须在锁内把数据拷贝出来**，或者用引用计数。

### LRU 链表方向别搞反

```c
static cache_block *cache_head;    /* 最近使用 */
static cache_block *cache_tail;    /* 最久未用 */
```

约定：

- 命中时 `unlink_block(b); link_front(b);` —— 移到头部
- 淘汰时摘尾部 `cache_evict_one()` 取 `cache_tail`

如果搞反了（头当最旧），会变成 FIFO 或者 MRU，缓存测试可能还能过（因为只测"能命中"），但真实场景下命中率会很低。

### `driver.sh` 的测试逻辑

三个测试各测什么，理解了才知道为什么这么设计：

**Basic**：拉 5 个文件经代理和直连对比

```text
home.html      文本
csapp.c        源码（长文本）
tiny.c         源码
godzilla.jpg   二进制图片
tiny           可执行文件（二进制，含大量 \0）
```

后两个专门测二进制安全。

**Concurrency**：先让一个请求撞上永不响应的 `nop-server`

```text
nop-server 接受连接后 spin，永不回应
  → 代理发出请求后卡在 Rio_readnb
  → 如果是顺序的，整个代理被这一个请求占死
  → 后续正常请求全部超时
  → 只有多线程/多进程才能通过
```

**Cache**：拉几个文件 → **杀掉 tiny** → 再拉一次

```text
Killing tiny
Fetching a cached copy of ./tiny/home.html into ./.noproxy
Success: Was able to fetch tiny/home.html from the cache.
```

服务器都死了还能返回内容，只能是缓存给的。

### 本次实测结果

```text
basicScore: 40/40
concurrencyScore: 15/15
cacheScore: 15/15
totalScore: 70/70
```

三部分全部满分。中间改过两次：

1. `P`/`V` 传 mutex 指针 → 改成 `pthread_mutex_lock`/`unlock`
2. `nop-server.py` shebang → 改成 `python3`

### 手动验证缓存

```bash
cd tiny && ./tiny 8000 &
cd .. && ./proxy 8080 &

curl -x http://localhost:8080 http://localhost:8000/home.html   # 第一次，miss
kill %1                                                          # 杀掉 tiny
curl -x http://localhost:8080 http://localhost:8000/home.html   # 第二次，应命中
```

第二次如果还能返回完整 HTML，缓存生效。

### `killall` 清理残留进程

driver 会起 tiny、proxy、nop-server 三个进程。中途失败可能留下孤儿：

```bash
killall -q proxy tiny nop-server.py 2>/dev/null
```

跑 driver 前先清理，避免端口占用导致莫明失败。

### 端口冲突

driver 随机选端口（`PORT_START=1024`，`MAX_RAND=63000`），但并发测试里三个服务同时起，可能撞上系统里已占用的端口。

```bash
ss -lntp | grep -E ":8080|:8000"
```

手动测试时指定明确端口，别用随机值。
