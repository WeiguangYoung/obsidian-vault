---
{"dg-publish":true,"permalink":"/csdn/Docker-容器-DevOps/Ubuntu安装docker开发环境/","title":"Ubuntu安装docker开发环境","tags":["docker","容器","运维"],"dg-note-properties":{"category":"Docker / 容器 / DevOps","title":"Ubuntu安装docker开发环境","source":"csdn","created":"2022-02-25","tags":["docker","容器","运维"],"url":"https://blog.csdn.net/weixin_45536921/article/details/123130686"}}
---


##### 1.使用apt安装docker和docker-compose

```
# 安装docker
sudo apt-get install docker.io
sudo groupadd docker
sudo gpasswd -a ${USER} docker
sudo systemctl restart docker

# 安装dokcer-compose
sudo apt-get install docker-compose

```

##### 2.更换国内源

修改```
/etc/docker/daemon.json
```,没有则创建


```
{
    "registry-mirrors":[
        "http://docker.mirrors.ustc.edu.cn",
        "http://hub-mirror.c.163.com",
        "http://registry.docker-cn.com"
    ] ,
    "insecure-registries":[
        "docker.mirrors.ustc.edu.cn",
        "registry.docker-cn.com"
    ]
}

```

##### 3.安装docker图像管理工具portainer

```
# 创建持久化卷
docker volume create portainer_data

# 后台启动项目
docker run -d -p 8000:8000 -p 9443:9443 --name portainer \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v portainer_data:/data \
    portainer/portainer-ce:2.11.1

```