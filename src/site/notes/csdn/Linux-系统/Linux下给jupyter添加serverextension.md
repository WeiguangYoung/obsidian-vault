---
{"dg-publish":true,"permalink":"/csdn/Linux-系统/Linux下给jupyter添加serverextension/","title":"Linux下给jupyter添加serverextension","tags":["python"],"dg-note-properties":{"category":"Linux / 系统","title":"Linux下给jupyter添加serverextension","source":"csdn","created":"2021-07-15","tags":["python"],"url":"https://blog.csdn.net/weixin_45536921/article/details/118753422"}}
---



### 自定义jupyter后台接口

首先确保已经安装了jupyter，如果没有则执行```
pip3 install jupyter
```


#### 1.安装自定义的python module

目录机构结构为


```
- setup.py
- my_module/
  - __init__.py

```
setup.py代码


```python
import setuptools

setuptools.setup(
    name='my_module',
    version='0.1',
    packages=['my_module'],
)

```
my_module/**init**.py代码


```python
from notebook.utils import url_path_join
from notebook.base.handlers import IPythonHandler


def _jupyter_server_extension_paths():
    return [{
        "module": "my_module"
    }]


class HelloWorldHandler(IPythonHandler):
    """接口请求，基于Tornado框架"""
    def get(self):
        self.finish('Hello, world!')


def load_jupyter_server_extension(nb_server_app):
    """安装拓展后，启动jupyter服务会被自动调用"""
    nb_server_app.log.info("my module enabled!")
    web_app = nb_server_app.web_app
    host_pattern = '.*
在终端执行


```shell
python setup.py install

```
出现下面的字样则表明安装成功


```
Installed /home/konkii/pyvirtualenv/lab_test/lib/python3.8/site-packages/my_module-0.1-py3.8.egg
Processing dependencies for my-module==0.1
Finished processing dependencies for my-module==0.1

```

#### 2.添加module到serverextension

```shell
jupyter serverextension enable --py my_module

```
出现下面的字样则表明安装成功


```
Enabling: my_module
- Writing config: /home/konkii/.jupyter
    - Validating...
      my_module  OK

```

#### 3.启动jupyter notebook

启动jupyter后，浏览器输入 http://127.0.0.1:8888/hello

出现 Hello, world!
    route_pattern = url_path_join(web_app.settings['base_url'], '/hello')
    web_app.add_handlers(host_pattern, [(route_pattern, HelloWorldHandler)])

```
在终端执行


{{CODE_BLOCK_4}}
出现下面的字样则表明安装成功


{{CODE_BLOCK_5}}

#### 2.添加module到serverextension

{{CODE_BLOCK_6}}
出现下面的字样则表明安装成功


{{CODE_BLOCK_7}}

#### 3.启动jupyter notebook

启动jupyter后，浏览器输入 http://127.0.0.1:8888/hello

出现 Hello, world!