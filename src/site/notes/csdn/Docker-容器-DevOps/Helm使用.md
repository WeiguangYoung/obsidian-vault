---
{"dg-publish":true,"permalink":"/csdn/Docker-容器-DevOps/Helm使用/","title":"Helm使用","tags":["容器"],"dg-note-properties":{"category":"Docker / 容器 / DevOps","title":"Helm使用","source":"csdn","created":"2022-10-30","tags":["容器"],"url":"https://blog.csdn.net/weixin_45536921/article/details/127598427"}}
---


## Helm使用


### 1.安装


#### 1.1.apt安装

```
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

```

#### 1.2.脚本安装

```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

```

#### 1.3.二进制版本安装

```
# 下载需要的版本
$ https://github.com/helm/helm/releases
# 解压压缩包
$ tar -zxvf helm-v3.0.0-linux-amd64.tar.gz
# 移动到需要的目录中
$ mv linux-amd64/helm /usr/local/bin/helm

```

### 2.常用命令

```
# 添加:有效的Helm-chart仓库
# 更新:确定可以拿到最新的charts列表
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add brigade https://brigadecore.github.io/charts
helm repo update

# 之后就可以查找相关的charts列表
helm search repo bitnami

# 了解到这个chart的基本信息
helm show chart bitnami/mysql
# 获取关于该chart的所有信息
helm show all bitnami/mysql

# 安装对应版本服务
helm install bitnami/mysql --generate-name

# 列出所有可被部署的版本
helm list

# 卸载一个版本
$ helm uninstall mysql-1612624192

# 查看帮助信息
$ helm get -h

```
基本命令


```
helm repo add bitnami https://charts.bitnami.com/bitnami	添加有效的 Helm-chart 仓库
helm repo list	查看配置的 chart 仓库
helm search repo wordpress	从添加的仓库中查找 chart 的名字
helm install happy-panda bitnami/wordpress	安装一个新的 helm 包
helm status happy-panda	来追踪展示 release 的当前状态
helm show values bitnami/wordpress	查看 chart 中的可配置选项
helm uninstall happy-panda	从集群中卸载一个 release
helm list	看到当前部署的所有 release
helm pull bitnami/wordpress	下载和查看一个发布的 chart
helm upgrade	升级 release 版本
helm rollback	恢复 release 版本

```

### 3.概念

Helm 安装 charts 到 Kubernetes 集群中，每次安装都会创建一个新的 release。


Chart
- Chart 代表着 Helm 包。
- 你可以把它看作是 Apt 或 Yum 在 Kubernetes 中的等价物。
- 它包含在 Kubernetes 集群内部运行应用程序，工具或服务所需的所有资源定义。


Repository
- Repository(仓库)是用来存放和共享 charts 的地方。
- 它就像 Fedora 的软件包仓库，只不过它是供 Kubernetes 包所使用的。


Release
- Release 是运行在 Kubernetes 集群中的 chart 的实例。
- 一个 chart 通常可以在同一个集群中安装多次，每一次安装都会创建一个新的 release。


Helm 按照以下顺序安装资源(这里列出主要的一些)：


```
	Namespace
	NetworkPolicy
	ResourceQuota
	LimitRange
	ServiceAccount
	Secret
	SecretList
	ConfigMap
	StorageClass
	PersistentVolume
	PersistentVolumeClaim
	Role
	RoleList
	RoleBinding
	RoleBindingList
	Service
	DaemonSet
	Pod
	ReplicationController
	ReplicaSet
	Deployment
	HorizontalPodAutoscaler
	StatefulSet
	Job
	CronJob
	Ingress
	APIService

```
参考```
https://mp.weixin.qq.com/s/LAaHqfU9mTLdu5ez8iI1Cw
```