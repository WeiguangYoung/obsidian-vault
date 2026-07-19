---
{"dg-publish":true,"permalink":"/knowledge/devops-general/通用DevOps知识体系/","tags":["DevOps","通用","面试","知识体系"],"dg-note-properties":{"date":"2026-07-19","tags":["DevOps","通用","面试","知识体系"]}}
---


# DevOps 知识体系

> 通用 DevOps 工程师知识体系，涵盖 CI/CD、容器编排、可观测性、SRE 等核心模块。

---

## 一、Linux & 网络基础 — [[knowledge/devops-general/DevOps-01-Linux与网络基础\|📄]]

| 模块 | 要点 |
|:----|:------|
| **Linux 系统管理** | 文件系统、权限、进程管理、systemd、用户与组 |
| **Shell 自动化** | Bash 脚本、awk/sed/jq 文本处理、定时任务 (cron) |
| **网络基础** | TCP/IP 协议栈、DNS、HTTP/HTTPS、负载均衡、CDN |
| **安全基础** | SSH、TLS/SSL、防火墙 (iptables/nftables)、SELinux/AppArmor |

## 二、CI/CD 持续集成交付 — [[knowledge/devops-general/DevOps-02-CICD持续集成\|📄]]

| 模块 | 要点 |
|:----|:------|
| **流水线平台** | Jenkins Pipeline (Groovy) / GitLab CI (.gitlab-ci.yml) / GitHub Actions |
| **流水线设计** | 编译 → 静态检查 → 单元测试 → 镜像构建 → 部署 → 验收测试 → 发布 |
| **多环境管理** | dev / test / staging / production 环境隔离与晋级策略 |
| **发布策略** | 蓝绿部署、金丝雀发布、滚动更新、A/B 测试 |
| **GitOps** | ArgoCD / Flux CD，以 Git 为单一事实源驱动部署 |

## 三、容器与编排 — [[knowledge/devops-general/DevOps-03-容器与编排\|📄]]

| 模块 | 要点 |
|:----|:------|
| **Docker** | Dockerfile（多阶段构建）、docker-compose、镜像优化、安全扫描 |
| **Kubernetes** | Pod / Deployment / Service / Ingress / ConfigMap / Secret / PVC |
| **调度与资源** | 亲和性、污点容忍、HPA / VPA、资源限制与 QoS |
| **网络与存储** | CNI (Calico/Flannel)、CSI、网络策略 |
| **Helm** | Chart 编写、values 分层、Chart Museum / Harbor |
| **多集群** | 集群联邦（KubeFed）、跨集群发布、多地域灾备 |

## 四、基础设施即代码 (IaC) — [[knowledge/devops-general/DevOps-04-IaC基础设施即代码\|📄]]

| 模块 | 要点 |
|:----|:------|
| **Terraform** | HCL 语法、Provider、State 管理、Workspace、Module 复用 |
| **Ansible** | Playbook、Role、Inventory、幂等执行 |
| **Packer** | 不可变基础设施镜像构建 |

## 五、制品与配置管理 — [[knowledge/devops-general/DevOps-05-制品与配置管理\|📄]]

| 模块 | 要点 |
|:----|:------|
| **代码管理** | Git 分支策略（Git Flow / Trunk-Based）、Code Review 流程 |
| **制品仓库** | Nexus / JFrog Artifactory（通用制品）、Harbor（容器镜像） |
| **配置管理** | 配置中心（Apollo / Nacos）、Secret 管理（Vault） |

## 六、质量门禁 — [[knowledge/devops-general/DevOps-06-质量门禁\|📄]]

| 模块 | 要点 |
|:----|:------|
| **静态分析** | SonarQube、Cppcheck、ESLint、Checkstyle |
| **单元测试** | GTest (C++)、JUnit (Java)、pytest (Python)、覆盖率报告 |
| **安全扫描** | Trivy（容器）、SAST / DAST、依赖漏洞扫描、SBOM |
| **行业专项** | 汽车：MISRA-C、ASPICE、ISO 26262；智驾：HIL/SIL、CANoe |

## 七、安全 — [[knowledge/devops-general/DevOps-07-安全\|📄]]

| 模块 | 要点 |
|:----|:------|
| **DevSecOps** | 安全左移、威胁建模、安全编码规范 |
| **供应链安全** | SBOM、依赖扫描、镜像签名、SLSA 框架 |
| **密钥管理** | Vault / 云 KMS、密钥轮转、最小权限 |
| **访问控制** | RBAC、LDAP/OIDC、堡垒机、审计日志 |
| **合规** | SOC2、ISO 27001、GDPR、等级保护 |

## 八、可观测性 — [[knowledge/devops-general/DevOps-08-可观测性\|📄]]

| 模块 | 要点 |
|:----|:------|
| **指标监控** | Prometheus（采集 + PromQL）、Grafana（Dashboard + 告警） |
| **日志管理** | ELK Stack / Loki / Splunk，日志采集 → 检索 → 告警 |
| **链路追踪** | Jaeger / OpenTelemetry / SkyWalking |
| **告警体系** | AlertManager 规则配置、告警分级、值班轮转、告警收敛 |

## 九、云平台 — [[knowledge/devops-general/DevOps-09-云平台\|📄]]

| 模块 | 要点 |
|:----|:------|
| **国内云** | 阿里云 ACK / 腾讯云 TKE / 火山引擎 VKE |
| **海外云** | AWS EKS / Azure AKS / GCP GKE |
| **多云管理** | 成本优化、跨云灾备、合规要求 |
| **云原生服务** | 对象存储（OSS/S3）、负载均衡（SLB/ELB）、CDN、Serverless |

## 十、脚本与开发 — [[knowledge/devops-general/DevOps-10-脚本与开发\|📄]]

| 语言 | 场景 |
|:----|:------|
| **Python** | 自动化运维脚本、CI/CD 工具开发、REST API 封装、数据分析 |
| **Shell (Bash)** | 构建脚本、系统巡检、日志处理、快速原型 |
| **Go** | K8s Operator、CLI 工具、高性能中间件 |
| **Groovy** | Jenkins Pipeline 共享库 |

## 十一、嵌入式构建（汽车/智驾行业特有） — [[knowledge/devops-general/DevOps-11-嵌入式构建\|📄]]

| 模块 | 要点 |
|:----|:------|
| **交叉编译** | arm-none-eabi-gcc、工具链配置、编译选项优化 |
| **构建系统** | Make / CMake / Bazel |
| **链接脚本** | MEMORY 分区、SECTIONS 布局、Bootloader 隔离 |
| **构建产物** | .elf → .bin / .hex 转换、固件签名与校验 |
| **ADAS 工具链** | HIL / SIL 测试环境对接、CANoe 自动化集成 |

## 十二、SRE 方法论 — [[knowledge/devops-general/DevOps-12-SRE方法论\|📄]]

- **SLO / SLI / Error Budget**：量化系统可靠性，指导发布节奏
- **故障演练**：混沌工程（Chaos Mesh / Litmus）
- **On-Call 机制**：事件分级、响应 SOP、事后复盘（Blameless Postmortem）
- **容量规划**：流量预估、资源弹性伸缩

---
