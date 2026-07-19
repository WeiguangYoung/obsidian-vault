---
{"dg-publish":true,"permalink":"/knowledge/devops/devops知识体系/04-IaC基础设施即代码/","tags":["DevOps","IaC","Terraform","Ansible","Packer"],"dg-note-properties":{"date":"2026-07-19","tags":["DevOps","IaC","Terraform","Ansible","Packer"]}}
---


# 四、基础设施即代码 (IaC)

> 手动操作服务器是事故的温床。IaC 把基础设施当作代码管理——版本化、可复现、可评审。

## 4.1 核心工具对比

| 工具 | 范式 | 特点 | 适用 |
|:----|:------|:------|:------|
| **Terraform** | 声明式 | 基础设施全生命周期，多云支持 | 云资源编排（创建/管理 K8s 集群、VPC、DB） |
| **Ansible** | 过程式 | 无 Agent，SSH 直连，幂等 | 服务器配置、应用部署、日常运维 |
| **Packer** | 声明式 | 构建不可变镜像 | 标准化 AMI / Docker 基础镜像 |

## 4.2 Terraform

### 核心概念

```
Provider（云厂商 API 适配）
    ↓
Resource（具体资源：VPC、ECS、RDS、ACK）
    ↓
State（当前基础设施状态文件）
    ↓
Plan（差异对比：期望 vs 实际）
    ↓
Apply（执行变更）
```

### 项目结构

```
terraform/
├── main.tf          # 资源定义
├── variables.tf     # 变量声明
├── outputs.tf       # 输出（IP、域名等）
├── terraform.tfvars # 变量值（不提交敏感信息）
└── versions.tf      # Provider 版本锁定
```

### main.tf 示例

```hcl
terraform {
  required_providers {
    alicloud = { source = "aliyun/alicloud", version = "~> 1.200" }
  }
}

provider "alicloud" {
  region = "cn-shanghai"
}

resource "alicloud_vpc" "main" {
  vpc_name   = "prod-vpc"
  cidr_block = "10.0.0.0/16"
}

resource "alicloud_vswitch" "subnet" {
  vpc_id     = alicloud_vpc.main.id
  cidr_block = "10.0.1.0/24"
  zone_id    = "cn-shanghai-b"
}
```

### 常用命令

```bash
terraform init          # 初始化 Provider
terraform plan          # 预览变更（不会执行）
terraform apply         # 执行变更
terraform destroy       # 销毁资源
terraform state list    # 查看当前状态管理的资源
```

### State 管理

| 方案 | 说明 |
|:----|:------|
| **本地** | 不适合团队协作 |
| **远程** | S3/OSS + DynamoDB 锁（推荐） |
| **Terraform Cloud** | 托管方案，Web UI + 团队协作 |

```hcl
# 远程 State（阿里云 OSS）
terraform {
  backend "oss" {
    bucket = "tf-state-prod"
    key    = "prod/terraform.tfstate"
    region = "cn-shanghai"
  }
}
```

### 模块复用

```hcl
# 定义模块
module "k8s" {
  source = "./modules/ack"
  cluster_name = "prod"
  vpc_id       = alicloud_vpc.main.id
  node_count   = 3
}

# 引用输出
module "app" {
  source    = "./modules/app"
  k8s_host  = module.k8s.cluster_endpoint
}
```

## 4.3 Ansible

### 核心概念

| 概念 | 说明 |
|:----|:------|
| **Inventory** | 主机清单（IP、分组、变量） |
| **Playbook** | YAML 格式的任务编排 |
| **Role** | 可复用的任务集合（目录结构标准化） |
| **Module** | 原子操作单元（yum、copy、service…） |
| **幂等** | 重复执行结果一致，不会重复安装 |

### Playbook 示例

```yaml
---
- name: 部署 Web 服务
  hosts: web
  become: yes
  vars:
    app_port: 8080

  tasks:
    - name: 安装 Nginx
      yum: name=nginx state=present

    - name: 复制配置
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: restart nginx

    - name: 启动服务
      service: name=nginx state=started enabled=yes

  handlers:
    - name: restart nginx
      service: name=nginx state=restarted
```

### Inventory 示例

```ini
[web]
10.0.1.10
10.0.1.11

[db]
10.0.2.10 ansible_user=dbadmin

[all:vars]
ansible_ssh_private_key_file=~/.ssh/prod_key
```

### Role 目录结构

```
roles/
└── nginx/
    ├── tasks/main.yml       # 任务
    ├── templates/nginx.conf.j2  # 模板
    ├── handlers/main.yml    # 回调
    └── vars/main.yml        # 变量
```

## 4.4 Packer

构建不可变镜像，确保所有实例基于相同镜像启动。

```hcl
# packer.pkr.hcl
source "alicloud-ecs" "base" {
  image_name    = "prod-base-{{timestamp}}"
  instance_type = "ecs.c6.large"
  region        = "cn-shanghai"
  ssh_username  = "root"
}

build {
  sources = ["source.alicloud-ecs.base"]

  provisioner "shell" {
    inline = [
      "yum update -y",
      "yum install -y docker nginx",
      "systemctl enable docker",
    ]
  }
}
```

## 4.5 IaC 最佳实践

- **代码化一切**：基础设施变更和其他代码一样走 MR → Review → Apply
- **DRY**：公共模块抽取复用，不要每个项目复制粘贴
- **State 远程管理**：绝不提交 tfstate 到 Git，用远程 Backend 存储
- **Plan 必看**：Apply 之前必须 Review Plan 输出
- **敏感变量**：用环境变量或 Vault，不写在 tfvars 里
