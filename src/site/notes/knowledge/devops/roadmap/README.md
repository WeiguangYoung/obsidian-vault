---
{"dg-publish":true,"permalink":"/knowledge/devops/roadmap/README/","tags":["DevOps","嵌入式","云原生","AIOps","知识体系"],"dg-note-properties":{"date":"2026-07-22","tags":["DevOps","嵌入式","云原生","AIOps","知识体系"]}}
---


# DevOps 知识体系

> 覆盖 Linux/网络 → 容器编排 → CI/CD → 制品与依赖 → 质量与安全 → 可观测性 → 平台工程 → AIOps 的完整知识体系。

---

## 一、Linux & 网络基础（含脚本与开发） — [[01-Linux与网络基础\|📄]]

| 模块 | 要点 |
|:----|:------|
| **Linux 系统管理** | 文件系统、权限、进程管理、systemd、用户与组 |
| **Shell 自动化** | Bash 脚本、awk/sed/jq 文本处理、定时任务 (cron) |
| **网络基础** | TCP/IP、DNS、HTTP/HTTPS、负载均衡 |
| **安全基础** | SSH、TLS/SSL、防火墙 (iptables/nftables) |
| **脚本与开发** | Python、Shell、Go、Groovy 语言选型与自动化脚本 |

## 二、容器与编排（含云平台） — [[knowledge/devops/roadmap/02-容器与编排\|📄]]

| 模块 | 要点 |
|:----|:------|
| **Docker** | Dockerfile 多阶段构建、docker-compose、镜像优化、安全扫描（Trivy） |
| **Kubernetes** | Pod / Deployment / Service / Ingress / ConfigMap / Secret / PVC |
| **调度与资源** | HPA、资源限制与 QoS、亲和性与反亲和 |
| **Helm** | Chart 编写、values 分层、Harbor 仓库 |
| **多集群** | 国内外多集群发布管理 |
| **云平台** | 阿里云 ACK、腾讯云 TKE、AWS EKS、多云管理与云原生服务 |

## 三、CI/CD 持续集成交付 — [[knowledge/devops/roadmap/03-CICD持续集成\|📄]]

| 模块 | 要点 |
|:----|:------|
| **流水线平台** | Jenkins Pipeline (Groovy)、GitLab CI、GitHub Actions |
| **流水线设计** | 拉取 → 编译 → 静态检查 → 单元测试 → 镜像构建 → 部署 → 报告 |
| **GitOps (ArgoCD)** | Application/ApplicationSet 多集群同步、自动漂移检测 |
| **发布策略** | 蓝绿部署、金丝雀发布、滚动更新 |
| **环境管理** | 环境层级、晋升流程、一致性、隔离 |
| **多仓库与评审** | Repo 多仓库管理、Gerrit 代码评审（+1/+2） |
| **容器平台** | KubeSphere 多租户、可视化流水线、灰度发布 |
| **多技术栈统一流水线** | C++/Java/Go 多语言项目统一 CI/CD 设计 |

## 四、制品与配置管理 — [[knowledge/devops/roadmap/04-制品与配置管理\|📄]]

| 模块 | 要点 |
|:----|:------|
| **代码管理** | Git 分支策略（Git Flow / Trunk-Based）、Code Review |
| **制品仓库** | Nexus（通用制品）、Harbor（容器镜像）、Jfrog Artifactory |
| **配置管理** | Secret 管理（Vault）、配置中心 |
| **项目管理集成** | 需求-代码-构建-测试追溯 |
| **Java 构建** | Maven（pom.xml/GAV/生命周期）、Gradle（Task/Wrapper） |
| **C++ 依赖管理** | Conan（conanfile.py）、vcpkg（Manifest 模式）、二进制缓存 |

## 五、质量与安全 — [[knowledge/devops/roadmap/05-质量门禁\|📄]]

| 模块 | 要点 |
|:----|:------|
| **静态分析** | Cppcheck、SonarQube、MISRA-C 检查 |
| **单元测试** | GTest (C++)、pytest (Python)、覆盖率报告 |
| **安全扫描** | Trivy（容器）、依赖漏洞扫描、SBOM |
| **行业专项** | MISRA-C、ASPICE 流程、功能安全 ISO 26262、HIL/SIL |
| **DevSecOps** | 安全左移、威胁建模、供应链安全（SLSA/Sigstore） |
| **密钥与访问控制** | Vault、RBAC、堡垒机、LDAP/OIDC |
| **合规** | SOC2 / ISO 27001 / 等级保护 / GDPR |

## 六、可观测性 — [[knowledge/devops/roadmap/06-可观测性\|📄]]

| 模块 | 要点 |
|:----|:------|
| **指标监控** | Prometheus（采集 + PromQL）、Grafana（Dashboard + 告警） |
| **日志管理** | ELK / Loki，日志采集 → 检索 → 告警 |
| **链路追踪** | Trace，分布式调用链 |
| **告警体系** | AlertManager、告警分级、值班轮转、告警收敛 |
| **效能度量** | DORA 四大指标、数据采集、Grafana 效能看板 |

## 七、AI 赋能 DevOps (AIOps) — [[knowledge/devops/roadmap/07-AIOps智能运维\|📄]]

| 模块 | 要点 |
|:----|:------|
| **AI Coding 提效** | GitHub Copilot / Cursor / Claude Code，辅助流水线与脚本编写 |
| **AI 代码评审** | MR/PR 自动审查、语义级规范检查、逻辑缺陷检测、自研架构 |
| **智能构建诊断** | 错误日志分类、根因分析（RCA）、修复建议生成、历史故障匹配 |
| **流水线智能优化** | 测试影响分析（TIA）、构建加速、资源调度优化、失败预测 |
| **大模型工程化** | API 调用 vs 私有化部署（vLLM/Ollama）、RAG 增强、数据安全 |
| **AIOps 落地路线** | Phase 1 代码评审 → Phase 2 构建诊断 → Phase 3 流水线优化 |

## 八、汽车领域知识（车载协议综述） — [[knowledge/devops/roadmap/08-汽车领域知识\|📄]]

| 模块 | 要点 |
|:----|:------|
| **车载总线** | CAN / CAN FD、LIN、FlexRay、车载以太网 |
| **诊断** | UDS（SID/DID/DTC/会话）、ISO-TP、DoIP、刷写流程 |
| **架构** | AUTOSAR CP / AP、RTE / BSW / DCM / DEM、SOA |
| **OTA** | SOTA / FOTA、差分包、双分区回滚、安全验签 |
| **车载以太网** | 100BASE-T1、SOME/IP、TSN、服务发现 |
| **汽车 DevOps** | 交叉编译、MISRA-C、Bootloader 制品、HIL/SIL 测试、OTA 流水线 |
