---
{"dg-publish":true,"permalink":"/knowledge/language/backend/05-Django入门/","title":"Django 入门","tags":["python","django","web","orm","mvt","全家桶"],"dg-note-properties":{"category":"后端 / Web 框架","title":"Django 入门","created":"2026-11-01","tags":["python","django","web","orm","mvt","全家桶"]}}
---


# Django 入门

> Django 是「**自带电池**」的 Python Web 框架：ORM、管理后台、认证、表单、迁移全都有。
> 相反，Flask 只给你路由。对照见 [[knowledge/language/backend/01-Flask入门\|Flask 入门]]。

## 一、和 Flask 的根本区别

| | Django | Flask |
|:----|:----|:----|
| 定位 | 全家桶（batteries included） | 微框架（micro） |
| ORM | **自带** Django ORM | 无（外挂 SQLAlchemy） |
| 数据迁移 | **自带** `makemigrations` / `migrate` | Flask-Migrate（Alembic） |
| 管理后台 | **自带** `/admin` | 无（flask-admin 扩展） |
| 认证权限 | **自带** auth + 权限模型 | 自己拼 |
| 表单校验 | **自带** Forms | 自己拼 / WTForms |
| 项目结构 | **强约定**（apps 目录） | 自由 |
| 适合 | 业务系统、后台管理、内容站 | API、微服务、内部小工具 |

> 一句话：**Django 给你整套骨架，你往里填；Flask 给你一块空地，你自己搭。**
> 国外 Django 用得极多（Instagram、Pinterest、Disqus 都是），国内偏向 Flask / FastAPI。

## 二、安装与创建项目

```bash
pip install django
django-admin --version

# 建项目（注意最后那个点，避免多套一层目录）
django-admin startproject mysite .

# 建应用
python manage.py startapp blog
```

**项目 vs 应用**

| 概念 | 含义 |
|:----|:----|
| project（项目） | 整个站点，装配置和路由总表 |
| app（应用） | 功能模块（blog / user / order），可跨项目复用 |

**标准目录**：

```
mysite/
├── manage.py
├── mysite/                # 项目配置包
│   ├── settings.py
│   ├── urls.py            # 根路由
│   ├── wsgi.py            # 生产入口（gunicorn 用）
│   └── asgi.py
└── blog/                  # 应用
    ├── models.py          # 数据模型
    ├── views.py           # 视图（业务逻辑）
    ├── urls.py            # 应用路由
    ├── admin.py           # 后台注册
    ├── apps.py
    └── migrations/        # 迁移脚本
```

## 三、MVT：Django 的核心设计

```
请求 → URLconf(urls.py) → View(views.py) → Model(models.py) ↔ DB
                                    ↓
                              Template(html)
```

| 层 | 文件 | 职责 |
|:----|:----|:----|
| **M**odel | `models.py` | 数据定义，一个类 = 一张表 |
| **V**iew | `views.py` | 处理请求、返回响应 |
| **T**emplate | `templates/` | 渲染 HTML（API 场景可跳过） |

> 和 MVC 的区别：Django 的 View 相当于 MVC 的 Controller，Template 相当于 View。

## 四、最小可跑流程（4 步）

### 4.1 注册应用

`mysite/settings.py`：

```python
INSTALLED_APPS = [
    # ... 自带的
    'blog',
]
```

### 4.2 写模型

`blog/models.py`：

```python
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField(blank=True)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['-created']

    def __str__(self):
        return self.title
```

### 4.3 迁移建表

```bash
python manage.py makemigrations     # 生成迁移脚本
python manage.py migrate            # 执行建表
```

> **铁律**：改完 `models.py` 必须走这两步。只改代码不迁移，表不会变。

### 4.4 路由 + 视图

`blog/urls.py`：

```python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.article_list, name='article_list'),
    path('<int:pk>/', views.article_detail, name='article_detail'),
]
```

`mysite/urls.py`：

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('blog/', include('blog.urls')),
]
```

`blog/views.py`：

```python
from django.shortcuts import render, get_object_or_404
from .models import Article

def article_list(request):
    articles = Article.objects.all()
    return render(request, 'blog/list.html', {'articles': articles})

def article_detail(request, pk):
    article = get_object_or_404(Article, pk=pk)
    return render(request, 'blog/detail.html', {'article': article})
```

### 4.5 跑起来

```bash
python manage.py runserver 0.0.0.0:8000
```

## 五、URL 路由

```python
from django.urls import path, re_path, include

urlpatterns = [
    path('articles/', include('blog.urls')),
    path('user/<int:uid>/', views.user),
    path('year/<int:year>/<slug:month>/', views.archive),   # 关键字参数
    re_path(r'^api/v1/.*

| 转换器 | 匹配类型 |
|:----|:----|
| `str` | 非空字符串（不含 `/`） |
| `int` | 整数 |
| `slug` | 字母数字 + `-` + `_` |
| `uuid` | UUID |
| `path` | 含 `/` 的字符串 |

**命名空间 + 反向解析**：

```python
# urls.py
path('blog/', include(('blog.urls', 'blog'), namespace='blog'))

# 模板中
{% url 'blog:article_detail' article.pk %}

# 代码中
from django.urls import reverse
reverse('blog:article_detail', args=[1])
```

## 六、ORM：最核心的差异点

### 6.1 查询

```python
Article.objects.all()
Article.objects.filter(title__icontains='python')      # LIKE '%python%'（忽略大小写）
Article.objects.exclude(status='draft')
Article.objects.get(pk=1)                              # 不存在抛 Article.DoesNotExist
Article.objects.order_by('-created')[:10]
Article.objects.count()
Article.objects.exists()
```

**字段查询后缀（`__` 之后）**

| 后缀 | 含义 |
|:----|:----|
| `exact` | 相等（默认） |
| `iexact` | 忽略大小写相等 |
| `contains` / `icontains` | 包含 / 忽略大小写 |
| `in` | 在列表中 |
| `gt` / `gte` / `lt` / `lte` | 比较 |
| `range` | 区间 |
| `isnull` | 是否为空 |
| `startswith` / `endswith` | 前缀 / 后缀 |
| `date` / `year` / `month` | 日期切片 |

```python
# 链式 + Q 对象做 OR
from django.db.models import Q, F, Count, Sum, Avg

Article.objects.filter(Q(title__icontains='django') | Q(content__icontains='django'))
Article.objects.filter(views__gte=F('likes'))                 # 字段间比较
Article.objects.values('category').annotate(n=Count('id'))     # 分组统计
```

### 6.2 增删改

```python
a = Article.objects.create(title='hello', content='...')   # 一步创建
a.title = 'hi'; a.save()                                   # 改
a.save(update_fields=['title'])                            # 只更新部分字段
Article.objects.filter(status='draft').update(status='archived')   # 批量改
a.delete()
```

### 6.3 外键与关联查询

```python
class Category(models.Model):
    name = models.CharField(max_length=50)

class Article(models.Model):
    category = models.ForeignKey(Category, on_delete=models.CASCADE, related_name='articles')
```

```python
a.category.name                       # 正向：文章 → 分类
c.articles.all()                      # 反向：related_name
Article.objects.filter(category__name='tech')      # 跨表查询
Article.objects.select_related('category')         # 避免 N+1（外键，JOIN）
Article.objects.prefetch_related('tags')           # 避免 N+1（多对多，二次查询）
```

> **N+1 问题是 Django 性能头号杀手**，记住 `select_related`（FK/OneToOne）与 `prefetch_related`（M2M/反向外键）。

## 七、管理后台（Django 最强卖点）

```bash
python manage.py createsuperuser
# 访问 /admin
```

`blog/admin.py`：

```python
from django.contrib import admin
from .models import Article, Category

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    list_display = ('id', 'title', 'category', 'created')
    list_filter = ('category', 'created')
    search_fields = ('title', 'content')
    list_per_page = 20

admin.site.register(Category)
```

**几分钟就能得到一个可增删改查、带权限、带搜索过滤的后台** —— 这是内部工具类需求的最优解。

## 八、模板

```python
render(request, 'blog/list.html', {'articles': articles})
```

```html
{% extends "base.html" %}
{% block content %}
  {% for a in articles %}
    <a href="{% url 'blog:article_detail' a.pk %}">{{ a.title }}</a>
    <small>{{ a.created|date:"Y-m-d" }}</small>
  {% empty %}
    <p>暂无数据</p>
  {% endfor %}
{% endblock %}
```

| 语法 | 作用 |
|:----|:----|
| `{{ var }}` | 输出（默认转义，防 XSS） |
| `{{ var\|safe }}` | 不转义（仅受信任内容） |
| `{% for %}` / `{% if %}` | 控制 |
| `{% extends %}` / `{% block %}` | 模板继承 |
| `{% url %}` | 反向解析 URL |
| `{% csrf_token %}` | **表单必须加**，否则 403 |

## 九、写 API：DRF（Django REST Framework）

```bash
pip install djangorestframework
```

```python
# settings.py
INSTALLED_APPS += ['rest_framework']
```

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Article

class ArticleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'created']
```

```python
# blog/views_api.py
from rest_framework import viewsets
from .models import Article
from .serializers import ArticleSerializer

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
```

```python
# urls.py
from rest_framework.routers import DefaultRouter
router = DefaultRouter()
router.register('articles', ArticleViewSet)
urlpatterns = [path('api/', include(router.urls))]
```

> 一个 ViewSet + Router，**list/create/retrieve/update/delete 五个接口全齐**。
> 纯 API 场景现在更流行 **FastAPI**，但 Django 生态成熟、后台强，内部平台仍首选。

## 十、Settings 与生产化

```python
# settings.py 关键项
DEBUG = False
ALLOWED_HOSTS = ['example.com']
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': os.getenv('DB_NAME'),
        'USER': os.getenv('DB_USER'),
        'PASSWORD': os.getenv('DB_PASSWORD'),   # 绝不写死
        'HOST': os.getenv('DB_HOST'),
        'PORT': '3306',
        'OPTIONS': {'charset': 'utf8mb4'},
    }
}
STATIC_ROOT = BASE_DIR / 'staticfiles'
```

```bash
python manage.py collectstatic --noinput
```

**生产部署**

```bash
pip install gunicorn
gunicorn mysite.wsgi:application -w 4 -b 0.0.0.0:8000
```

| 组件 | 作用 |
|:----|:----|
| gunicorn | WSGI 服务 |
| nginx | 反代 + 静态文件 |
| PostgreSQL/MySQL | 生产库（**别用 SQLite**，并发写会报 `database is locked`） |
| 健康检查 | `/healthz` 供 k8s 探针 |

### 附：`OperationalError: database is locked` 怎么解

SQLite 是文件级锁，**只允许一个写事务**。多个线程/进程同时写时，后来的那个超时等待锁释放，超时后抛：

```
django.db.utils.OperationalError: database is locked
```

Python 的 SQLite 包装器有个**默认超时值**，决定第二个线程最多等多久。三种处理方式（按推荐度排）：

**① 换库（根本解，生产必做）**

```python
# MySQL
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': os.getenv('DB_NAME'),
        'USER': os.getenv('DB_USER'),
        'PASSWORD': os.getenv('DB_PASSWORD'),
        'HOST': os.getenv('DB_HOST', '127.0.0.1'),
        'PORT': '3306',
        'OPTIONS': {'charset': 'utf8mb4'},
    }
}

# PostgreSQL（更推荐，行级锁 + MVCC，并发写表现最好）
# pip install psycopg2-binary
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME'),
        'USER': os.getenv('DB_USER'),
        'PASSWORD': os.getenv('DB_PASSWORD'),
        'HOST': os.getenv('DB_HOST', '127.0.0.1'),
        'PORT': '5432',
    }
}
```

> SQLite 到某个阶段对真实应用来说就「过于精简」了，并发报错就是信号。

**② 加超时（缓解，不改架构）**

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(YOUR_DIR, 'db.sqlite3'),
        'OPTIONS': {
            'timeout': 20,        # 秒，默认 5
            # 更优：WAL 模式，读写不再互斥
            'init_command': 'PRAGMA journal_mode=WAL;',
        },
    }
}
```

> `timeout` 只是**让等待更久**，不会真正解决；**`WAL` 模式才是 SQLite 单机读写并存的关键**（写仍串行，但读不再被写阻塞）。

**③ 减少并发 / 缩短事务**

```python
# 反例：事务里做慢 I/O，锁一持就是几秒
with transaction.atomic():
    obj = Model.objects.get(pk=1)
    requests.get('https://slow-api')      # ← 千万别在事务里
    obj.save()

# 正例：先做 I/O，再开事务
resp = requests.get('https://slow-api')
with transaction.atomic():
    Model.objects.filter(pk=1).update(v=resp.json()['v'])
```

| 场景 | 建议 |
|:----|:----|
| 开发/单用户小工具 | SQLite + WAL，够用 |
| 多进程（gunicorn `-w 4`） | **必须换 MySQL/PostgreSQL** |
| 容器化部署 | 数据库独立服务，绝不与 app 同容器 |

**补充排查**：数据库文件权限也会导致锁（尤其多用户/容器卷挂载场景）：

```bash
ls -l db.sqlite3          # 确认 app 用户有读写权限
chmod 666 db.sqlite3      # 临时放宽（仅本地调试，生产不要）
```

> 权限问题在容器里尤其常见：宿主机的 `db.sqlite3` 属主是 root，容器里以非 root 跑就必然 `database is locked`/`unable to open database file`。正确做法是 `chown` 到容器运行用户，别 `chmod 666`。

## 十一、常用命令速查

```bash
python manage.py runserver            # 开发服务器
python manage.py makemigrations       # 生成迁移
python manage.py migrate              # 执行迁移
python manage.py createsuperuser      # 建超管
python manage.py shell                # 交互式 shell（可跑 ORM）
python manage.py dbshell              # 进数据库
python manage.py collectstatic        # 收集静态文件
python manage.py test                 # 跑测试
python manage.py showmigrations       # 查看迁移状态
python manage.py startapp xxx         # 新建应用
```

## 十二、常见坑

| 现象 | 原因 | 解法 |
|:----|:----|:----|
| `no such table` | 没迁移 | `makemigrations` + `migrate` |
| 表单 POST 返回 403 | 缺 CSRF token | 模板加 `{% csrf_token %}` |
| `DisallowedHost` | 未配 `ALLOWED_HOSTS` | 加上域名/`*`（仅调试） |
| 页面 500 但无堆栈 | `DEBUG=False` | 临时开 DEBUG 或看日志 |
| 接口慢 | N+1 查询 | `select_related` / `prefetch_related` |
| 静态文件 404 | 未 collectstatic | `collectstatic` + nginx 配置 |
| 中文显示 `乱码` | 库编码不是 utf8mb4 | 建库时指定 + `OPTIONS charset` |
| `database is locked` | SQLite 并发写 | 换 MySQL/PostgreSQL；临时可开 WAL（见「Settings 与生产化」附录） |

## 十三、一句话总结

> Django = **ORM + 迁移 + Admin + 认证** 四件套开箱即用；
> 先写 `models` → `makemigrations/migrate` → 写 `views` → 挂 `urls`；写 API 上 DRF，生产用 gunicorn。

## 相关文档

- [[knowledge/language/backend/01-Flask入门\|Flask 入门]]
- [[knowledge/language/mysql/01-MySQL数据库基本命令\|MySQL 数据库基本命令]]
- [[knowledge/language/mysql/01-MySQL数据库基本命令\|MySQL 数据库基本命令]]
, views.api),                     # 正则（少用）
]
```

| 转换器 | 匹配类型 |
|:----|:----|
| `str` | 非空字符串（不含 `/`） |
| `int` | 整数 |
| `slug` | 字母数字 + `-` + `_` |
| `uuid` | UUID |
| `path` | 含 `/` 的字符串 |

**命名空间 + 反向解析**：

{{CODE_BLOCK_11}}

## 六、ORM：最核心的差异点

### 6.1 查询

{{CODE_BLOCK_12}}

**字段查询后缀（`__` 之后）**

| 后缀 | 含义 |
|:----|:----|
| `exact` | 相等（默认） |
| `iexact` | 忽略大小写相等 |
| `contains` / `icontains` | 包含 / 忽略大小写 |
| `in` | 在列表中 |
| `gt` / `gte` / `lt` / `lte` | 比较 |
| `range` | 区间 |
| `isnull` | 是否为空 |
| `startswith` / `endswith` | 前缀 / 后缀 |
| `date` / `year` / `month` | 日期切片 |

{{CODE_BLOCK_13}}

### 6.2 增删改

{{CODE_BLOCK_14}}

### 6.3 外键与关联查询

{{CODE_BLOCK_15}}

{{CODE_BLOCK_16}}

> **N+1 问题是 Django 性能头号杀手**，记住 `select_related`（FK/OneToOne）与 `prefetch_related`（M2M/反向外键）。

## 七、管理后台（Django 最强卖点）

{{CODE_BLOCK_17}}

`blog/admin.py`：

{{CODE_BLOCK_18}}

**几分钟就能得到一个可增删改查、带权限、带搜索过滤的后台** —— 这是内部工具类需求的最优解。

## 八、模板

{{CODE_BLOCK_19}}

{{CODE_BLOCK_20}}

| 语法 | 作用 |
|:----|:----|
| `{{ var }}` | 输出（默认转义，防 XSS） |
| `{{ var\|safe }}` | 不转义（仅受信任内容） |
| `{% for %}` / `{% if %}` | 控制 |
| `{% extends %}` / `{% block %}` | 模板继承 |
| `{% url %}` | 反向解析 URL |
| `{% csrf_token %}` | **表单必须加**，否则 403 |

## 九、写 API：DRF（Django REST Framework）

{{CODE_BLOCK_21}}

{{CODE_BLOCK_22}}

{{CODE_BLOCK_23}}

{{CODE_BLOCK_24}}

{{CODE_BLOCK_25}}

> 一个 ViewSet + Router，**list/create/retrieve/update/delete 五个接口全齐**。
> 纯 API 场景现在更流行 **FastAPI**，但 Django 生态成熟、后台强，内部平台仍首选。

## 十、Settings 与生产化

{{CODE_BLOCK_26}}

{{CODE_BLOCK_27}}

**生产部署**

{{CODE_BLOCK_28}}

| 组件 | 作用 |
|:----|:----|
| gunicorn | WSGI 服务 |
| nginx | 反代 + 静态文件 |
| PostgreSQL/MySQL | 生产库（**别用 SQLite**，并发写会报 `database is locked`） |
| 健康检查 | `/healthz` 供 k8s 探针 |

### 附：`OperationalError: database is locked` 怎么解

SQLite 是文件级锁，**只允许一个写事务**。多个线程/进程同时写时，后来的那个超时等待锁释放，超时后抛：

{{CODE_BLOCK_29}}

Python 的 SQLite 包装器有个**默认超时值**，决定第二个线程最多等多久。三种处理方式（按推荐度排）：

**① 换库（根本解，生产必做）**

{{CODE_BLOCK_30}}

> SQLite 到某个阶段对真实应用来说就「过于精简」了，并发报错就是信号。

**② 加超时（缓解，不改架构）**

{{CODE_BLOCK_31}}

> `timeout` 只是**让等待更久**，不会真正解决；**`WAL` 模式才是 SQLite 单机读写并存的关键**（写仍串行，但读不再被写阻塞）。

**③ 减少并发 / 缩短事务**

{{CODE_BLOCK_32}}

| 场景 | 建议 |
|:----|:----|
| 开发/单用户小工具 | SQLite + WAL，够用 |
| 多进程（gunicorn `-w 4`） | **必须换 MySQL/PostgreSQL** |
| 容器化部署 | 数据库独立服务，绝不与 app 同容器 |

**补充排查**：数据库文件权限也会导致锁（尤其多用户/容器卷挂载场景）：

{{CODE_BLOCK_33}}

> 权限问题在容器里尤其常见：宿主机的 `db.sqlite3` 属主是 root，容器里以非 root 跑就必然 `database is locked`/`unable to open database file`。正确做法是 `chown` 到容器运行用户，别 `chmod 666`。

## 十一、常用命令速查

{{CODE_BLOCK_34}}

## 十二、常见坑

| 现象 | 原因 | 解法 |
|:----|:----|:----|
| `no such table` | 没迁移 | `makemigrations` + `migrate` |
| 表单 POST 返回 403 | 缺 CSRF token | 模板加 `{% csrf_token %}` |
| `DisallowedHost` | 未配 `ALLOWED_HOSTS` | 加上域名/`*`（仅调试） |
| 页面 500 但无堆栈 | `DEBUG=False` | 临时开 DEBUG 或看日志 |
| 接口慢 | N+1 查询 | `select_related` / `prefetch_related` |
| 静态文件 404 | 未 collectstatic | `collectstatic` + nginx 配置 |
| 中文显示 `乱码` | 库编码不是 utf8mb4 | 建库时指定 + `OPTIONS charset` |
| `database is locked` | SQLite 并发写 | 换 MySQL/PostgreSQL；临时可开 WAL（见「Settings 与生产化」附录） |

## 十三、一句话总结

> Django = **ORM + 迁移 + Admin + 认证** 四件套开箱即用；
> 先写 `models` → `makemigrations/migrate` → 写 `views` → 挂 `urls`；写 API 上 DRF，生产用 gunicorn。

## 相关文档

- [[knowledge/language/backend/01-Flask入门\|Flask 入门]]
- [[knowledge/language/mysql/01-MySQL数据库基本命令\|MySQL 数据库基本命令]]
- [[knowledge/language/mysql/01-MySQL数据库基本命令\|MySQL 数据库基本命令]]
