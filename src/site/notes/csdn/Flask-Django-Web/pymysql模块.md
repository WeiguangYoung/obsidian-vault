---
{"dg-publish":true,"permalink":"/csdn/Flask-Django-Web/pymysql模块/","title":"pymysql模块","tags":["mysql","数据库","python"],"dg-note-properties":{"category":"Flask / Django / Web","title":"pymysql模块","source":"csdn","created":"2021-07-05","tags":["mysql","数据库","python"],"url":"https://blog.csdn.net/weixin_45536921/article/details/118493946"}}
---



pymysql是一个第三方库，如果自己的计算机上没有可以在终端使用命令进行安装。


```
sudo pip3 install pymysql

```
- pymysql使用流程

- 建立数据库连接：db = pymysql.connect(…)
- 创建游标对象：cur = db.cursor()
- 游标方法:  cur.execute(“insert …”)
- 提交到数据库或者获取数据 :  db.commit() / cur.fetchall()
- 关闭游标对象 ：cur.close()
- 断开数据库连接 ：db.close()

- 常用函数

```python
db = pymysql.connect(参数列表)
功能: 链接数据库

host ：主机地址,本地 localhost
port ：端口号,默认3306
user ：用户名
password ：密码
database ：库
charset ：编码方式,推荐使用 utf8

```
```
cur = db.cursor() 
功能： 创建游标
返回值：返回游标对象,用于执行具体SQL命令

```
```
cur.execute(sql,args_list) 
功能： 执行SQL命令
参数： sql sql语句
      args_list  列表，用于给sql语句传递参量
      
cur.executemany(sql命令,args_list)
功能： 多次执行SQL命令，执行次数由列表中元组数量决定
参数： sql sql语句
      args_list  列表中包含元组 每个元组用于给sql语句传递参量，一般用于写操作。

```
```python
db.commit() 提交到数据库执行，必须支持事务操作才有效
db.rollback() 回到原来的数据形态，必须支持事务操作才有效
cur.fetchone() 获取查询结果集的第一条数据，查找到返回一个元组否则返回None
cur.fetchmany(n) 获取前n条查找到的记录，返回结果为元组嵌套元组， ((记录1),(记录2))，查询不到内容返回空元组。
cur.fetchall() 获取所有查找到的记录，返回结果形式同上。

```
```python
cur.close() 关闭游标对象
db.close() 关闭数据库连接

```
文件存储
存储文件路径
- 优点：节省数据库空间，提取方便
- 缺点：文件或者数据库发生迁移会导致文件丢失


存储文件本身
- 优点：安全可靠，数据库在文件就在
- 缺点：占用数据库空间大，文件存取效率低