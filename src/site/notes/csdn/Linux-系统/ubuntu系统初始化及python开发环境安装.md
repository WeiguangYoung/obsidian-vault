---
{"dg-publish":true,"permalink":"/csdn/Linux-系统/ubuntu系统初始化及python开发环境安装/","title":"ubuntu系统初始化及python开发环境安装","tags":["ubuntu","linux","运维"],"dg-note-properties":{"category":"Linux / 系统","title":"ubuntu系统初始化及python开发环境安装","source":"csdn","created":"2021-12-13","tags":["ubuntu","linux","运维"],"url":"https://blog.csdn.net/weixin_45536921/article/details/121900301"}}
---


## ubuntu20.04初始化


### 1.修改apt源(以阿里源为例)

备份

```
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
```

修改源文件

```
sudo vi /etc/apt/sources.list
```

写入以下内容


```
deb http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse

```

### 2.安装pip3及虚拟环境

安装pip3


```
sudo apt-get install python3-pip

```
更换pip源(以阿里源为例)


```
mkdir ~/.pip
cd ~/.pip
vi pip.conf

```
写入以下内容


```
[global]
index-url = http://mirrors.aliyun.com/pypi/simple/
 
[install]
trusted-host=mirrors.aliyun.com

```
安装python虚拟环境


```
pip3 install virtualenv

```
如果出现以下错误


```
WARNING: The script virtualenv is installed in ‘/home/xxx/.local/bin’ which is not on PATH.
Consider adding this directory to PATH or, if you prefer to suppress this warning, use --no-warn-script-location.

```
执行


```
# 此处换成自己的路径
echo 'export PATH=/home/xxx/.local/bin:$PATH'  >> ~/.bashrc
source ~/.bashrc

# 重新安装
pip3 uninstall virtualenv
pip3 install virtualenv

```

### 3.安装docker和docker-compose

```
# 安装docker
sudo apt-get install docker.io
sudo groupadd docker
sudo gpasswd -a ${USER} docker
sudo systemctl restart docker

# 安装dokcer-compose
sudo apt-get install docker-compose

```
docker换源


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

### 4.其他常用工具

git

```
sudo apt install git
```

ssh

```
sudo apt install openssh-server
```


清华源


```
apt:  https://mirrors.tuna.tsinghua.edu.cn
pip:  https://pypi.tuna.tsinghua.edu.cn/simple

```