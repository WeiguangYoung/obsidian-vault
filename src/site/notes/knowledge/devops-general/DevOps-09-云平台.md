---
{"dg-publish":true,"permalink":"/knowledge/devops-general/DevOps-09-云平台/","tags":["DevOps","云平台","K8s","AWS","阿里云","多云"],"dg-note-properties":{"date":"2026-07-19","tags":["DevOps","云平台","K8s","AWS","阿里云","多云"]}}
---


# 六、云平台

## 6.1 主流云厂商

| 云厂商 | K8s 服务 | 特点 |
|:----|:------|:------|
| **阿里云** | ACK（容器服务） | 国内份额第一，生态全 |
| **腾讯云** | TKE | 游戏/社交场景有优势 |
| **火山引擎** | VKE | 字节生态，性价比高 |
| **AWS** | EKS | 全球份额第一，服务最丰富 |
| **Azure** | AKS | 企业 Microsoft 生态 |
| **GCP** | GKE | K8s 原生体验最好 |

## 6.2 阿里云 ACK 核心服务

| 服务 | 说明 |
|:----|:------|
| **ACK** | 托管 K8s，支持专有版/托管版/边缘版 |
| **ACR** | 容器镜像服务（对标 Harbor，企业版支持镜像扫描） |
| **SLB** | 负载均衡（对接 Ingress / Service LoadBalancer） |
| **OSS** | 对象存储（静态资源、日志归档） |
| **NAS** | 文件存储（多 Pod 共享存储） |
| **SLS** | 日志服务（采集 + 检索 + 告警） |
| **CMS** | 云监控（对接 Prometheus） |

## 6.3 AWS EKS 核心服务

| 服务 | 说明 |
|:----|:------|
| **EKS** | 托管 K8s 控制面 |
| **ECR** | 容器镜像仓库 |
| **ELB** | 负载均衡（ALB Ingress Controller） |
| **S3** | 对象存储 |
| **CloudWatch** | 监控 + 日志 |
| **Route53** | DNS 服务（多集群流量调度） |

## 6.4 多云管理

### 多集群架构

```
                 ┌──────────┐
                 │  DNS/GSLB│
                 └────┬─────┘
           ┌──────────┴──────────┐
           ▼                     ▼
   ┌──────────────┐    ┌──────────────┐
   │  阿里云上海    │    │  AWS 新加坡   │
   │  ACK + OSS   │    │  EKS + S3    │
   └──────────────┘    └──────────────┘
```

### 关键考量

| 维度 | 策略 |
|:----|:------|
| **流量调度** | DNS/GSLB 按地域分流，就近接入 |
| **数据同步** | 异步复制或数据库内置多地域 |
| **CI/CD** | ArgoCD + ApplicationSet 统一管理多集群 |
| **监控** | 多集群统一 Prometheus / Grafana |
| **成本** | 按需选择不同厂商的实例类型和计费方式 |
| **灾备** | 主备或双活，RPO/RTO 定义清晰 |

## 6.5 云原生服务

| 类型 | 厂商方案 | 开源方案 |
|:----|:------|:------|
| **对象存储** | OSS / S3 / COS | MinIO |
| **负载均衡** | SLB / ELB / CLB | Nginx / HAProxy |
| **CDN** | 全站加速 / CloudFront | 自建不推荐 |
| **Serverless** | 函数计算 (FC) / Lambda | Knative / OpenFaaS |
| **消息队列** | RocketMQ / SQS | Kafka / RabbitMQ |
| **数据库** | RDS / Aurora | 自建 MySQL / PostgreSQL |
