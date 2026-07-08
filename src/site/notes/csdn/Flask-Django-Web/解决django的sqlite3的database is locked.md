---
{"dg-publish":true,"permalink":"/csdn/Flask-Django-Web/解决django的sqlite3的database is locked/","title":"解决django的sqlite3的database is locked","tags":["python","django","sqlite3"],"dg-note-properties":{"category":"Flask / Django / Web","title":"解决django的sqlite3的database is locked","source":"csdn","created":"2021-06-28","tags":["python","django","sqlite3"],"url":"https://blog.csdn.net/weixin_45536921/article/details/118308364"}}
---


#### OperationalError: database is locked

SQLite 是一个轻量级数据库，因此不能支持高并发。错误表明您的应用程序遇到的并发性超出了默认配置中的处理能力。

这个错误意味着一个线程或进程在数据库连接上有一个排他锁，另一个线程超时等待锁被释放。

Python的SQLite包装器有一个默认超时值，用于确定第二个线程在超时并引发错误之前允许在锁定上等待多长时间。


如果您收到此错误，可以通过以下方式解决：


- 切换到另一个数据库后端。在某一点上，SQLite 对于实际应用程序来说变得过于“精简”了，而这些并发错误表明您已经达到了那个点。
- 重写代码以减少并发并确保数据库事务是短暂的。
- 通过设置timeout数据库选项增加默认超时值：

```
DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.sqlite3', 
            'NAME':   os.path.join(YOUR_DIR,'db.sqlite3'),     
            'OPTIONS': {
                'timeout': 20,
            }
        }
    }

```
这将使 SQLite 在抛出“数据库被锁定”错误之前等待更长的时间；它不会真正解决它们。


如果还不行的话，看看db.sqlite3的文件权限，改改看行不行

```
chmod 666 db.sqlite3
```