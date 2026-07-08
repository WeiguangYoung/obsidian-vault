---
{"dg-publish":true,"permalink":"/csdn/Docker-容器-DevOps/使用docker自定义nginx服务/","title":"使用docker自定义nginx服务","tags":["nginx","docker"],"dg-note-properties":{"category":"Docker / 容器 / DevOps","title":"使用docker自定义nginx服务","source":"csdn","created":"2021-12-30","tags":["nginx","docker"],"url":"https://blog.csdn.net/weixin_45536921/article/details/122242955"}}
---


使用docker自定义nginx服务


Dockerfile文件


```
FROM nginx
COPY nginx.conf /etc/nginx/conf.d/default.conf

# 此处更换为自己需要移动的文件或文件夹
COPY XXX /usr/share/nginx/html/

```
nginx.conf文件


```
server {
    listen 80;
    server_name _;
    client_max_body_size 1000m;
    autoindex on;
    autoindex_exact_size off;
    autoindex_localtime on;
    charset utf-8;

    location / {
      root /usr/share/nginx/html;
      try_files $uri $uri/ /index.html;
      autoindex on;
    }
}

```
构建镜像

```
docker build -t nginx:demo .
```

启动服务

```
docker run -d -p 80:80 nginx:demo
```

访问服务

```
http://127.0.0.1
```


#### k8s nginx demo

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: test-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-nginx
  template:
    metadata:
      labels:
        app: test-nginx
    spec:
      containers:
      - name: test-nginx
        image: nginx
        ports:
        - containerPort: 80
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: test-nginx
spec:
  selector:
    app: test-nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort


```