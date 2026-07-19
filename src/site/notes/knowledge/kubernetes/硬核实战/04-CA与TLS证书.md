---
{"dg-publish":true,"permalink":"/knowledge/kubernetes/硬核实战/04-CA与TLS证书/","tags":["Kubernetes","K8s","硬核实战"],"dg-note-properties":{"date":"2026-07-19","tags":["Kubernetes","K8s","硬核实战"]}}
---


# 配置 CA 与生成 TLS 证书

本实验使用 OpenSSL 搭建 PKI（公钥基础设施），创建证书颁发机构（CA），并为以下组件生成 TLS 证书：kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy。本节命令均在 `jumpbox` 上执行。

## 证书颁发机构（CA）

创建 CA 用于签发其他 K8s 组件的 TLS 证书。为简化流程，项目已提供 `ca.conf` 配置文件，定义了各组件证书的生成细节。

浏览配置文件：

```bash
cat ca.conf
```

不要求完全理解 `ca.conf`，但可以将其作为学习 OpenSSL 证书管理的起点。

每个 CA 始于一个私钥和根证书。此处我们创建自签名 CA——虽然对学习来说够用，但生产环境不应这样做。

生成 CA 配置、证书和私钥：

```bash
{
  openssl genrsa -out ca.key 4096
  openssl req -x509 -new -sha512 -noenc \
    -key ca.key -days 3653 \
    -config ca.conf \
    -out ca.crt
}
```

结果文件：

```txt
ca.crt ca.key
```

## 生成客户端与服务端证书

为各 K8s 组件及 `admin` 用户生成证书。

生成证书签名请求和签名证书：

```bash
certs=(
  "admin" "node-0" "node-1"
  "kube-proxy" "kube-scheduler"
  "kube-controller-manager"
  "kube-api-server"
  "service-accounts"
)
```

```bash
for i in ${certs[*]}; do
  openssl genrsa -out "${i}.key" 4096

  openssl req -new -key "${i}.key" -sha256 \
    -config "ca.conf" -section ${i} \
    -out "${i}.csr"

  openssl x509 -req -days 3653 -in "${i}.csr" \
    -copy_extensions copyall \
    -sha256 -CA "ca.crt" \
    -CAkey "ca.key" \
    -CAcreateserial \
    -out "${i}.crt"
done
```

查看生成的文件：

```bash
ls -1 *.crt *.key *.csr
```

## 分发证书

将证书拷贝到每台机器对应的路径。在生产环境中，这些证书应视为敏感凭据——因为 K8s 组件用它们互相认证。

分发到 Worker 节点（node-0 和 node-1）：

```bash
for host in node-0 node-1; do
  ssh root@${host} mkdir /var/lib/kubelet/

  scp ca.crt root@${host}:/var/lib/kubelet/

  scp ${host}.crt \
    root@${host}:/var/lib/kubelet/kubelet.crt

  scp ${host}.key \
    root@${host}:/var/lib/kubelet/kubelet.key
done
```

分发到控制节点（server）：

```bash
scp \
  ca.key ca.crt \
  kube-api-server.key kube-api-server.crt \
  service-accounts.key service-accounts.crt \
  root@server:~/
```

> `kube-proxy`、`kube-controller-manager`、`kube-scheduler` 和 `kubelet` 的客户端证书将在下一实验中用于生成客户端认证配置文件。

下一步：[生成 K8s 认证配置文件](05-kubernetes-configuration-files.md)
