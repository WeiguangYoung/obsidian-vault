---
{"dg-publish":true,"permalink":"/csdn/Linux-系统/Ubuntu系统golang开发环境搭建/","title":"Ubuntu系统golang开发环境搭建","tags":["ubuntu","golang","linux"],"dg-note-properties":{"category":"Linux / 系统","title":"Ubuntu系统golang开发环境搭建","source":"csdn","created":"2022-02-18","tags":["ubuntu","golang","linux"],"url":"https://blog.csdn.net/weixin_45536921/article/details/122997485"}}
---


### ubuntu golang安装


#### 1.进入go语言中文网```
https://studygolang.com/dl
```，查看自己需要的版本


#### 2.下载并解压

```
wget https://dl.google.com/go/go1.16.14.linux-amd64.tar.gz
sudo tar -zxvf go1.16.14.linux-amd64.tar.gz -C /usr/local 

```

#### 3.加入环境变量

修改profile

```
vim $HOME/.profile
```

写入以下内容


```
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin

```
激活profile

```
source $HOME/.profile
```


检查

```
go version
```


#### 4.修改代理

```
export GOPROXY=https://goproxy.cn
```

或

```
go env -w GOPROXY=https://goproxy.io,direct
```


#### 5.跨平台编译

```
SET CGO_ENABLED=0  // 禁用CGO
SET GOOS=linux  // 目标平台是linux
SET GOARCH=amd64  // 目标处理器架构是amd64

```
如：```
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build
```