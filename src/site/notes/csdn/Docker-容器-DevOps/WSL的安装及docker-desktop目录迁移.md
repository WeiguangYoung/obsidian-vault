---
{"dg-publish":true,"permalink":"/csdn/Docker-容器-DevOps/WSL的安装及docker-desktop目录迁移/","title":"WSL的安装及docker-desktop目录迁移","tags":["docker","ubuntu","windows"],"dg-note-properties":{"category":"Docker / 容器 / DevOps","title":"WSL的安装及docker-desktop目录迁移","source":"csdn","created":"2022-06-18","tags":["docker","ubuntu","windows"],"url":"https://blog.csdn.net/weixin_45536921/article/details/125352641"}}
---


### 安装WSL

微软官方文档：```
https://docs.microsoft.com/en-us/windows/wsl/install
```


查看wsl版本

```
wsl --list --online
``` 或者 ```
wsl -l -o
```


选择wsl1或者wsl2，推荐wsl2

```
wsl --set-default-version <Version#>
```

其中 ```
<Version#> 
``` 为 ```
1
``` 或者 ```
2
```


选择安装需要的版本，以下方式二选一

1.设置默认版本，然后安装默认版本

```
wsl -s <Distribution Name>
``` 或者 ```
wsl --setdefault <Distribution Name
```>

```
wsl --install
```

2.直接安装指定的版本

```
wsl --install -d <Distribution Name>
```


### 修改ubuntu和docker-desktop安装目录

ubuntu/docker-desktop/docker-desktop-data 修改方式一致


```
# 查看分发版列表 
wsl -l -v --all

# 关闭wsl
wsl --shutdown

# 迁移docker-desktop-data
wsl --export docker-desktop-data D:\Docker\docker-desktop-data.tar
wsl --unregister docker-desktop-data
wsl --import docker-desktop-data D:\Docker\docker-desktop-data\ D:\Docker\docker-desktop-data.tar --version 2

# 迁移docker-desktop
wsl --export docker-desktop D:\Docker\docker-desktop.tar
wsl --unregister docker-desktop
wsl --import docker-desktop D:\Docker\docker-desktop\ D:\Docker\docker-desktop.tar --version 2

# 迁移Ubuntu  这里替换成自己的版本
wsl --export Ubuntu D:\Docker\Ubuntu.tar
wsl --unregister Ubuntu
wsl --import Ubuntu D:\Docker\Ubuntu\ D:\Docker\Ubuntu.tar --version 2

# 再次查看分发版列表
wsl -l -v --all

```
注：清理无用的tar文件