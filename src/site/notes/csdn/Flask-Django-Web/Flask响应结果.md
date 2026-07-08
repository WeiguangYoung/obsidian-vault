---
{"dg-publish":true,"permalink":"/csdn/Flask-Django-Web/Flask响应结果/","title":"Flask响应结果","tags":["python","flask"],"dg-note-properties":{"category":"Flask / Django / Web","title":"Flask响应结果","source":"csdn","created":"2021-04-12","tags":["python","flask"],"url":"https://blog.csdn.net/weixin_45536921/article/details/115639221"}}
---



Flask的视图函数可返回三个参数，响应内容, 响应码, 响应头。大体有四种情况：


1.响应内容

2.响应内容, 响应码

3.响应内容, 响应头

4.响应内容, 响应码, 响应头


显然这里的重点是**响应内容**，就视图函数的返回值而言，官方给的定义是可以返回

```
a string, dict, tuple, Response instance, or WSGI callable
```


### 1.响应内容为字符(节)串，字典，会被自动转换成Response对象


返回值类型转换字符(节)串以该字符为响应体、响应码200、响应类型为text/html的响应对象。字典以该字典为响应体、响应码200、响应类型为application/json的响应对象。代码示例：```
from flask import Flask, make_response

app = Flask(__name__)
@app.route('/hello')
def hello():
    # 分别返回字符串，字节串，字典
    # return "Hello Flask"
    # return b"Hello Flask"
    return { "text": "Hello Flask" }

```

### 2.响应内容为元组，即上面提到2,3,4的三种情况

关于响应码和响应头不太了解的可以去先学习一下HTTP协议相关知识，响应码和响应头可以理解成响应对象的两个属性。

如果响应内容是字符(节)串/字典，和上面一样被自动转换成Response对象。

常见构造Response对象的方法有两种：


2.1.```
jsonify()
```

如果响应数据为一个列表，如用户列表信息，则可以使用jsonify()转换成Response对象，当然你也可以手动将字典转换成响应对象，常用于前后端分离项目。


代码示例：


```
from flask import Flask, jsonify

app = Flask(__name__)
@app.route("/users")
def users_api():
    users = ["Alice", "Bob", "Cindy"]
    return jsonify(users)
    # 你也可以自定义响应头或响应码，如
    # return jsonify(users), 200

```

2.2.```
make_response()
```

函数make_response的参数，和上文提到的视图函数返回值基本一致，不过响应内容只能是```
字符(节)串/字典
```，否则就成了套娃了，哈哈~~


2.2.1.返回值为字典，代码示例：


```
from flask import Flask, make_response

app = Flask(__name__)
@app.route('/hello')
def hello():
    # 响应头 
    headers = {
        'content-type': 'application/json'
    }
    user_details = {
        "username": "Alice"
    }
    response = make_response(user_details)
    return response, 200, headers

```
2.2.2.返回值为html文本，代码示例：


```
from flask import Flask, make_response

app = Flask(__name__)
@app.route('/hello')
def hello():
    # 支持html语法
    response = make_response('## hello')
    return response, 200

```
2.2.3.返回值为html模板，代码示例：


```
from flask import Flask, make_response, render_template

app = Flask(__name__)
@app.route('/hello')
def hello():
    # 自定义模板
    temp = render_template('hello.html')
    # 实际上传入还是字符串，render_template帮我们读取了这个文件
    response = make_response(temp)
    return response, 200

```
也可以直接通过修改make_response()返回值的形式，设置响应头和响应码


```
from flask import Flask, make_response

app = Flask(__name__)
@app.route('/hello')
def hello():
    headers = {
        'content-type':'text/plain'
    }
    response = make_response("", 404)
    response.headers = headers
    # 也可以这样写
    # response.headers["content-type"] = "text/plain"

    return response

```
关于响应码的格式，可以为符合规范的数字，如200, 400, 也可以是 ```
"200 OK"
```，但是注意，开头一定是数字，然后空格，再是自定义的说明。