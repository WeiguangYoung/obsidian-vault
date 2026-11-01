---
{"dg-publish":true,"permalink":"/knowledge/language/backend/01-Flask入门/","title":"Flask 入门","tags":["python","flask","web","路由","restful"],"dg-note-properties":{"category":"后端 / Web 框架","title":"Flask 入门","source":"csdn","created":"2021-04-08","updated":"2026-11-01","tags":["python","flask","web","路由","restful"],"url":"https://blog.csdn.net/weixin_45536921/article/details/115525919"}}
---


# Flask 入门

> 从「三行代码起一个 Web 服务」到「路由、请求、响应、蓝图」的完整最小闭环。
> 数据库接入见 [[knowledge/language/backend/03-PyMySQL模块\|PyMySQL 模块]] 与 [[knowledge/language/backend/04-Flask数据库迁移\|Flask 数据库迁移]]。

## 一、Flask 是什么

Flask 是 Python 的**微框架**（micro-framework）：核心只有路由和请求分发，其他全靠扩展。

| | Flask | Django |
|:----|:----|:----|
| 定位 | 微框架，按需组装 | 全家桶，开箱即用 |
| ORM | 无（SQLAlchemy 可选） | 自带 Django ORM |
| 管理后台 | 无（flask-admin） | 自带 |
| 适合 | 小服务、API、原型、嵌入到已有系统 | 中大型业务站 |

> 做**纯 API 服务**（给 CI/CD 平台写回调、做内部工具）时 Flask 极快上手，这是它在 DevOps 场景里的主要用途。

## 二、最小示例

```bash
pip install flask
```

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def index():
    return 'Hello Flask'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

```bash
python app.py
curl http://127.0.0.1:5000/
```

**要点**
- `debug=True` 开热重载 + 详细报错 —— **只能本地用，绝不能进生产**
- `host='0.0.0.0'` 才能被外部访问（默认只监听 127.0.0.1）

## 三、路由

### 3.1 基本规则

```python
@app.route('/user')                              # GET 默认
@app.route('/user', methods=['POST', 'PUT'])     # 显式声明方法
@app.route('/user/<int:uid>')                    # 类型转换器
@app.route('/file/<path:p>')                     # 允许包含 /
```

| 转换器 | 匹配 |
|:----|:----|
| `string` | 默认，不含 `/` |
| `int` | 整数 |
| `float` | 浮点数 |
| `path` | 含 `/` 的字符串 |
| `uuid` | UUID 字符串 |

### 3.2 获取路由参数

```python
@app.route('/user/<int:uid>')
def user(uid):
    return f'user {uid}'
```

### 3.3 URL 反向生成（`url_for`）

```python
from flask import url_for

@app.route('/user/<int:uid>')
def user(uid):
    return 'ok'

with app.test_request_context():
    print(url_for('user', uid=1))   # /user/1
```

> 模板和重定向里**一律用 `url_for`**，别写死路径 —— 改路由时不会全站崩。

## 四、获取请求参数

### 4.1 查询参数（query string）

```python
from flask import request

@app.route('/search')
def search():
    kw = request.args.get('kw', '')          # ?kw=xxx
    page = request.args.get('page', 1, type=int)
    tags = request.args.getlist('tag')       # 同名多值
    return f'{kw} {page} {tags}'
```

### 4.2 表单

```python
@app.route('/login', methods=['POST'])
def login():
    username = request.form.get('username')
    pwd = request.form.get('password')
    return f'{username}'
```

### 4.3 JSON 请求体

```python
@app.route('/api/job', methods=['POST'])
def create_job():
    data = request.get_json(silent=True) or {}     # silent=True 解析失败返回 None
    name = data.get('name')
    if not name:
        return {'code': 400, 'msg': 'name required'}, 400
    return {'code': 0, 'data': {'name': name}}
```

### 4.4 常用 request 属性

| 属性 | 说明 |
|:----|:----|
| `request.method` | 请求方法 |
| `request.args` | URL 查询参数（ImmutableMultiDict） |
| `request.form` | 表单数据 |
| `request.get_json()` | JSON 请求体 |
| `request.files` | 上传文件 |
| `request.headers` | 请求头 |
| `request.cookies` | Cookie |
| `request.remote_addr` | 客户端 IP |
| `request.url` / `path` | 完整 URL / 路径 |

## 五、响应

### 5.1 三种返回方式

```python
@app.route('/a')
def a():
    return 'text'                                  # 字符串 → text/html

@app.route('/b')
def b():
    return {'code': 0, 'msg': 'ok'}                # dict → application/json（Flask 1.1+）

@app.route('/c')
def c():
    return '<h1>hi</h1>', 201, {'X-Trace': 'abc'}  # (body, status, headers)
```

### 5.2 显式构造

```python
from flask import jsonify, make_response, redirect, url_for, abort

@app.route('/api/x')
def x():
    resp = make_response(jsonify(code=0, data=[]))
    resp.status_code = 200
    resp.headers['X-Custom'] = 'v'
    resp.set_cookie('k', 'v', httponly=True)
    return resp

@app.route('/go')
def go():
    return redirect(url_for('index'))

@app.route('/nope')
def nope():
    abort(404, description='resource not found')
```

### 5.3 错误处理

```python
@app.errorhandler(404)
def not_found(e):
    return {'code': 404, 'msg': str(e)}, 404

@app.errorhandler(Exception)
def on_error(e):
    app.logger.exception('unhandled')
    return {'code': 500, 'msg': 'internal error'}, 500
```

### 5.4 文件上传与下载

```python
import os
from flask import request, send_from_directory, send_file
from werkzeug.utils import secure_filename

UPLOAD_DIR = 'uploads'
os.makedirs(UPLOAD_DIR, exist_ok=True)

@app.route('/upload', methods=['POST'])
def upload():
    f = request.files.get('file')
    if not f:
        return {'code': 400, 'msg': 'no file'}, 400
    name = secure_filename(f.filename)      # 必须：防 ../ 路径穿越
    f.save(os.path.join(UPLOAD_DIR, name))
    return {'code': 0, 'name': name}

@app.route('/download/<name>')
def download(name):
    return send_from_directory(UPLOAD_DIR, name, as_attachment=True)
```

## 六、模板渲染（Jinja2）

```python
from flask import render_template

@app.route('/hello/<name>')
def hello(name):
    return render_template('hello.html', name=name)
```

`templates/hello.html`：

```html
<!doctype html>
<title>Hello</title>
<h1>Hello {{ name }}</h1>
{% if name == 'admin' %}<p>管理员</p>{% endif %}
<ul>
{% for item in items | default([]) %}
  <li>{{ item }}</li>
{% endfor %}
</ul>
```

| 语法 | 作用 |
|:----|:----|
| `{{ var }}` | 输出（自动 HTML 转义，防 XSS） |
| `{% ... %}` | 控制语句 |
| `{{ var \| safe }}` | 取消转义（**只在受信任内容上用**） |
| `{% extends %}` / `{% block %}` | 模板继承 |

> 纯 API 服务用不到模板；要给内部平台做个简单页面时才用。

## 七、配置管理

```python
class Config:
    SECRET_KEY = 'change-me'
    DEBUG = False
    SQLALCHEMY_DATABASE_URI = 'mysql+pymysql://u:p@127.0.0.1/db'

app.config.from_object(Config)
app.config.from_envvar('APP_CONFIG', silent=True)    # 生产用环境变量覆盖
```

**铁律**：`SECRET_KEY` 和数据库密码**不能写死在代码里**，用环境变量注入。

## 八、蓝图（Blueprint）：项目变大后的必然选择

单文件撑不住时，按业务拆蓝图：

```
app/
├── __init__.py         # create_app()
├── config.py
├── api/
│   ├── __init__.py
│   └── job.py          # bp = Blueprint('job', __name__)
└── web/
    └── views.py
```

```python
# app/api/job.py
from flask import Blueprint, jsonify

bp = Blueprint('job', __name__, url_prefix='/api/job')

@bp.route('/list')
def job_list():
    return jsonify(code=0, data=[])
```

```python
# app/__init__.py
from flask import Flask

def create_app():
    app = Flask(__name__)
    from .api.job import bp as job_bp
    app.register_blueprint(job_bp)
    return app
```

**工厂函数 + 蓝图**是 Flask 项目的标准骨架，测试和配置切换都靠它。

## 九、部署（别用 `app.run` 上生产）

```bash
# gunicorn：Linux 生产首选
pip install gunicorn
gunicorn -w 4 -b 0.0.0.0:8000 'app:create_app()'
```

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:8000", "app:create_app()"]
```

| 组件 | 作用 |
|:----|:----|
| gunicorn | WSGI 服务器，多 worker |
| nginx | 反向代理、静态文件、TLS |
| 健康检查 | `/healthz` 返回 200，供 k8s 探针用 |

> `app.run()` 是开发用的单进程服务器，**并发差、无进程管理**，绝不能上生产。

## 十、常见坑

| 现象 | 原因 | 解法 |
|:----|:----|:----|
| 改代码不生效 | 没开 debug / 多 worker | 本地 `debug=True`；生产重启 worker |
| 外部访问不了 | 只监听 127.0.0.1 | `host='0.0.0.0'` |
| `get_json()` 返回 415 | Content-Type 不是 json | 请求带 `Content-Type: application/json` |
| 中文返回 `\uXXXX` | jsonify 默认 ASCII | `app.json.ensure_ascii = False`（Flask 2.3+） |
| 上传文件名带 `../` | 未过滤 | `secure_filename()` |
| 生产内存泄漏 | debug 模式运行 | 生产关 debug |

## 十一、一句话总结

> `Flask(__name__)` 起服务 → `@app.route` 定路由 → `request` 取参 → `return` 决定响应 → 项目变大用**蓝图 + 工厂函数** → 生产用 **gunicorn**。

## 相关文档

- [[knowledge/language/backend/02-Flask响应结果\|Flask 响应结果]]
- [[knowledge/language/backend/03-PyMySQL模块\|PyMySQL 模块]]
- [[knowledge/language/backend/04-Flask数据库迁移\|Flask 数据库迁移]]
