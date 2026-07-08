---
{"dg-publish":true,"permalink":"/csdn/Flask-Django-Web/Flask数据库迁移/","title":"Flask数据库迁移","tags":["flask","python"],"dg-note-properties":{"category":"Flask / Django / Web","title":"Flask数据库迁移","source":"csdn","created":"2021-06-15","tags":["flask","python"],"url":"https://blog.csdn.net/weixin_45536921/article/details/117919769"}}
---



### Flask数据库迁移

数据库迁移工具安装


```shell
pip3 install Flask-Migrate==2.7.0 Flask-Script==2.0.6 Flask-SQLAlchemy==2.5.1

```
创建管理文件manage.py


```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_script import Manager, Server
from flask_migrate import Migrate, MigrateCommand


app = Flask(__name__)

# 初始化数据库对象
db = SQLAlchemy()

# 数据库对象初始化app
db.init_app(app)

# 设置管理器对象
manager = Manager(app)

# 配置数据库迁移
Migrate(app=app, db=db)

# 自定义命令
manager.add_command('db', MigrateCommand)  # 创建数据库映射命令 init/migrate/upgrade
manager.add_command('runserver', Server(host="0.0.0.0", port=5000, use_debugger=True))  # 创建启动命令

# 启动管理器
manager.run()

```
数据库迁移操作


```shell
python3 manage.py db init     # 初始化数据库迁移
python3 manage.py db migrate  # 生成数据库迁移文件
python3 manage.py db upgrade  # 执行迁移文件

```
启动项目


```shell
python3 manage.py runserver

```