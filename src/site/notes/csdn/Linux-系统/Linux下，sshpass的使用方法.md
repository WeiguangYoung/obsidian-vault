---
{"dg-publish":true,"permalink":"/csdn/Linux-系统/Linux下，sshpass的使用方法/","title":"Linux下，sshpass的使用方法","tags":["linux","后端"],"dg-note-properties":{"category":"Linux / 系统","title":"Linux下，sshpass的使用方法","source":"csdn","created":"2021-04-12","tags":["linux","后端"],"url":"https://blog.csdn.net/weixin_45536921/article/details/115630227"}}
---


ssh辅助工具，不用单独输入密码就可以连接ssh，一行命令连接服务器

不过不建议在公网环境下使用


## 1.在线安装

yum安装: ```
yum install sshpass
```

apt安装: ```
apt-get install sshpass
```

alpine安装: ```
apk add sshpass
```


## 2.示例

连接ssh: ```
sshpass -p xxx ssh root@192.168.11.11
```

常见错误:


报错：```
sshpass: Failed to run command: No such file or directory
```

检查ssh是否安装成功，直接执行```
ssh root@192.168.11.11
```，没有的话自行安装 openssh-client
没有任何反应

该命令需要先建立连接，直接用ssh连接发现需要输入yes/no

执行命令```
sshpass -p xxx ssh -o StrictHostKeyChecking=no root@192.168.11.11
``` 应该就ok了


使用scp: ```
sshpass -p xxx scp root@192.168.11.11:/root/test.txt test.txt
```

当然也可以在远程服务器执行你想执行的命令


```
sshpass -p xxx ssh root@192.168.11.11 {YOUR_COMMAND}

```
如:

```
sshpass -p xxx ssh root@192.168.11.11 ls
```

```
sshpass -p xxx ssh root@192.168.11.11 pwd
```