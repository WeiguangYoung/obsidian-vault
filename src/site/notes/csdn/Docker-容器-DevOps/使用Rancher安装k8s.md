---
{"dg-publish":true,"permalink":"/csdn/Docker-容器-DevOps/使用Rancher安装k8s/","title":"使用Rancher安装k8s","tags":["docker","容器","运维"],"dg-note-properties":{"category":"Docker / 容器 / DevOps","title":"使用Rancher安装k8s","source":"csdn","created":"2022-02-25","tags":["docker","容器","运维"],"url":"https://blog.csdn.net/weixin_45536921/article/details/123130798"}}
---


#### 使用Rancher安装k8s

提前安装docker


遇到的问题


```
# sudo apt install ufw
sudo ufw disable
sudo ufw status

```
```
# sudo apt install firewalld
sudo systemctl stop firewalld

```
```
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
iptables -F

```

##### 使用docker安装Rancher

```
# 在宿主机上创建Rancher的挂载目录：
mkdir -p /docker_volume/rancher_home/rancher
mkdir -p /docker_volume/rancher_home/auditlog

# 启动 rancher 容器
docker run -d --restart=unless-stopped -p 8080:80 -p 8443:443 \
 -v /docker_volume/rancher_home/rancher:/var/lib/rancher \
 -v /docker_volume/rancher_home/auditlog:/var/log/auditlog \

```