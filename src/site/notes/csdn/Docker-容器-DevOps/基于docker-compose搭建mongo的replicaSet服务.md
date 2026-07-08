---
{"dg-publish":true,"permalink":"/csdn/Docker-容器-DevOps/基于docker-compose搭建mongo的replicaSet服务/","title":"基于docker-compose搭建mongo的replicaSet服务","tags":["docker","ubuntu","容器"],"dg-note-properties":{"category":"Docker / 容器 / DevOps","title":"基于docker-compose搭建mongo的replicaSet服务","source":"csdn","created":"2022-03-31","tags":["docker","ubuntu","容器"],"url":"https://blog.csdn.net/weixin_45536921/article/details/123869637"}}
---


#### 基于docker-compose搭建mongo的replicaSet服务


##### 1.备份和恢复

备份

```
mongodump -h 127.0.0.1:27017 -u root -p 123456 -o ./dump
```

恢复

```
mongorestore -h 127.0.0.1:27017 -u root -p 123456  ./dump
```

注意：多replicaSet需要在Primary节点恢复


##### 2.docker-compose示例

```
version: '3.3'

services:
  mongodb1:
    image: mongo:4.2.1
    volumes:
      - ./data/mongo1:/data/db
    user: root
    container_name: mongodb1
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=123456
    ports:
      - 37017:27017
    command: mongod --replSet mongos 
    restart: always
    entrypoint:
      - bash
      - -c
      - |
        exec docker-entrypoint.sh $@

  mongodb2:
    image: mongo:4.2.1
    volumes:
      - ./data/mongo2:/data/db
    user: root
    container_name: mongodb2
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=123456
    ports:
      - 37018:27017
    command: mongod --replSet mongos 
    restart: always
    entrypoint:
      - bash
      - -c
      - |
        exec docker-entrypoint.sh $@

  mongodb3:
    image: mongo:4.2.1
    volumes:
      - ./data/mongo3:/data/db
    user: root
    container_name: mongodb3
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=123456
    ports:
      - 37019:27017
    command: mongod --replSet mongos
    restart: always
    entrypoint:
      - bash
      - -c
      - |
        exec docker-entrypoint.sh $@

```

##### 3.集群初始化

```
# 初始化节点
rs.initiate({
    _id: "mongos",
    members: [
        { _id : 0, host : "IP:37017" },
        { _id : 1, host : "IP:37018" },
        { _id : 2, host : "IP:37019" }
    ]
});

```

###### 其他常用命令

```
# 查看集群状态
rs.status()

# 检查是否为主节点
rs.isMaster()

添加节点
rs.add("mongodb1:27017")

# 移除节点
rs.remove("mongodb1:27017"); 

# 获取节点配置
cfg = rs.conf()

# 修改节点
cfg.members[0].host="mongodb1:27017"

# 重新配置
rs.reconfig(cfg) 

# 强制重新配置
rs.reconfig(cfg, {force : true})

```