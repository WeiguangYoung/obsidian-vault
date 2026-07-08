---
{"dg-publish":true,"permalink":"/csdn/Docker-容器-DevOps/Ubuntu系统Docker使用nvidia显卡/","title":"Ubuntu系统Docker使用nvidia显卡","tags":["docker","ubuntu","容器"],"dg-note-properties":{"category":"Docker / 容器 / DevOps","title":"Ubuntu系统Docker使用nvidia显卡","source":"csdn","created":"2021-12-15","tags":["docker","ubuntu","容器"],"url":"https://blog.csdn.net/weixin_45536921/article/details/121946079"}}
---


## Ubuntu系统Docker使用GPU


#### 1.配置nvidia源

```
curl -s -L https://nvidia.github.io/nvidia-container-runtime/gpgkey | \
  sudo apt-key add -

```
```
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)

```
```
curl -s -L https://nvidia.github.io/nvidia-container-runtime/$distribution/nvidia-container-runtime.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-runtime.list

```

#### 2.使用apt安装nvidia-container-runtime

```
sudo apt-get update
sudo apt-get install nvidia-container-runtime

```

#### 3.重启docker,并把runtime添加到docker中

```
dockerd --add-runtime=nvidia=/usr/bin/nvidia-container-runtime

```

#### 4.创建容器

创建容器时指定gpus参数


```
docker run -it --gpus=all 镜像名  执行命令

```
如：```
docker run -it --gpus=all nvidia/cuda:10.0-cudnn7-devel-ubuntu18.04 bash
```

进入容器后执行，如果正常会有GPU信息


```
nvidia-smi

```
docker-compose示例,指定runtime为nvidia


```
version: '2.3'
services:
  backend:
    image: XXX
    runtime: nvidia

```

#### 5.错误信息

创建容器错误1:

```
docker: Error response from daemon: could not select device driver "" with capabilities: [[gpu]]. ERRO[0000] error waiting for container: context canceled
```


```
sudo apt-get install nvidia-container-toolkit
systemctl restart docker

```
创建容器错误2:

```
docker: Error response from daemon: Unknown runtime specified nvidia. See 'docker run --help'.
```

检查/etc/docker下daemon.json,不存在则创建


```
{
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}

```
重启docker


```
sudo systemctl daemon-reload
sudo systemctl restart docker

```