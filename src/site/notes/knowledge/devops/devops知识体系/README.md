---
{"dg-publish":true,"permalink":"/knowledge/devops/devops知识体系/README/","tags":["DevOps","嵌入式","云原生","面试","知识体系","AIOps"],"dg-note-properties":{"date":"2026-07-22","tags":["DevOps","嵌入式","云原生","面试","知识体系","AIOps"]}}
---


# DevOps 知识体系

> 面向嵌入式 CI/CD + K8s 云原生 + 质量门禁 + 监控运维 + AIOps 的 DevOps 知识体系。

---

## 一、Linux & 网络基础 — [[knowledge/devops/devops知识体系/01-Linux与网络基础\|📄]]

| 模块 | 要点 |
|:----|:------|
| **Linux 系统管理** | 文件系统、权限、进程管理、systemd、用户与组 |
| **Shell 自动化** | Bash 脚本、awk/sed/jq 文本处理、定时任务 (cron) |
| **网络基础** | TCP/IP、DNS、HTTP/HTTPS、负载均衡 |
| **安全基础** | SSH、TLS/SSL、防火墙 (iptables/nftables) |

## 二、容器与编排 — [[knowledge/devops/devops知识体系/02-容器与编排\|📄]]

| 模块 | 要点 |
|:----|:------|
| **Docker** | Dockerfile 多阶段构建、docker-compose、镜像优化、安全扫描（Trivy） |
| **Kubernetes** | Pod / Deployment / Service / Ingress / ConfigMap / Secret / PVC |
| **调度与资源** | HPA、资源限制与 QoS、亲和性与反亲和 |
| **Helm** | Chart 编写、values 分层、Harbor 仓库 |
| **多集群** | 国内外多集群发布管理 |

## 三、CI/CD 持续集成交付 — [[knowledge/devops/devops知识体系/03-CICD持续集成\|📄]]

| 模块                  | 要点                                                                  |
| :------------------ | :------------------------------------------------------------------ |
| **流水线平台**           | Jenkins Pipeline (Groovy)、GitLab CI (.gitlab-ci.yml)、GitHub Actions |
| **流水线设计**           | 拉取 → 编译 → 静态检查 → 单元测试 → 镜像构建 → 部署 → 测试报告                            |
| **GitOps (ArgoCD)** | Application/ApplicationSet 多集群同步、自动漂移检测、Helm/Kustomize 集成           |
| **发布策略**            | 蓝绿部署、金丝雀发布（Argo Rollouts）、滚动更新                                      |
| **环境管理**            | Docker 构建镜像、License 管理、构建节点管理、交叉编译工具链环境                             |
| **嵌入式 CICD**        | CMake/Bazel 构建、交叉编译流水线、.elf → .bin/.hex 制品管理                        |

## 四、制品与配置管理 — [[knowledge/devops/devops知识体系/04-制品与配置管理\|📄]]

| 模块         | 要点                                               |
| :--------- | :----------------------------------------------- |
| **代码管理**   | Git 分支策略（Git Flow / Trunk-Based）、Code Review     |
| **制品仓库**   | Nexus (C/C++ 制品)、Harbor (容器镜像)、Jfrog Artifactory |
| **项目管理集成** | 需求-代码-构建-测试追溯                                    |
| **配置管理**   | Secret 管理（Vault）、配置中心                            |

## 五、质量门禁 — [[knowledge/devops/devops知识体系/05-质量门禁\|📄]]

| 模块        | 要点                                |
| :-------- | :-------------------------------- |
| **静态分析**  | Cppcheck、SonarQube、MISRA-C 检查     |
| **单元测试**  | GTest (C++)、pytest (Python)、覆盖率报告 |
| **安全扫描**  | Trivy（容器）、依赖漏洞扫描、SBOM             |
| **汽车专项**  | MISRA-C、ASPICE 流程、功能安全 ISO 26262  |
| **嵌入式测试** | HIL/SIL 测试对接、CANoe 自动化集成          |

## 六、可观测性 — [[knowledge/devops/devops知识体系/06-可观测性\|📄]]

| 模块 | 要点 |
|:----|:------|
| **指标监控** | Prometheus（采集 + PromQL）、Grafana（Dashboard + 告警） |
| **日志管理** | ELK / Loki，日志采集 → 检索 → 告警 |
| **告警体系** | AlertManager、告警分级、值班轮转、告警收敛 |
| **K8s 运维监控** | 集群健康、Pod 资源、网络策略、节点状态 |

## 七、云平台 — [[knowledge/devops/devops知识体系/07-云平台\|📄]]

| 模块 | 要点 |
|:----|:------|
| **国内云** | 阿里云 ACK、腾讯云 TKE、火山引擎 VKE |
| **多云管理** | 跨云灾备、多集群发布协调 |
| **云原生服务** | 对象存储（OSS）、负载均衡（SLB）、CDN |

## 八、脚本与开发 — [[knowledge/devops/devops知识体系/08-脚本与开发\|📄]]

| 语言 | 场景 |
|:----|:------|
| **Python** | 自动化运维脚本、CI/CD 工具开发、REST API 封装 |
| **Shell (Bash)** | 构建脚本、系统巡检、日志处理 |
| **Go** | K8s Operator、CLI 工具 |
| **Groovy** | Jenkins Pipeline 共享库 |

## 九、安全 — [[knowledge/devops/devops知识体系/09-安全\|📄]]

| 模块 | 要点 |
|:----|:------|
| **DevSecOps** | 安全左移、镜像签名、依赖扫描 |
| **供应链安全** | SBOM、SLSA 框架、漏洞管理 |
| **访问控制** | RBAC、LDAP/OIDC、堡垒机、审计日志 |
| **开源合规** | 开源许可证扫描、代码合规检查 |

## 十、自动化工具链与平台 — [[knowledge/devops/devops知识体系/10-自动化工具链与平台\|📄]]

| 模块 | 要点 |
|:----|:------|
| **自研 DevOps 平台** | 平台架构设计、插件化流水线、统一门户 |
| **机器人工具链** | 仿真平台集成、数据处理管道、可视化工具、自动化测试框架 |
| **AI Coding** | AI 辅助开发工具（GitHub Copilot / Cursor），提升团队效率 |
| **IaC 基础** | Ansible 自动化配置 |
| **GitOps** | ArgoCD 多集群部署、ApplicationSet、自动同步、Drift Detection |

## 十一、AI 赋能 DevOps (AIOps) 🆕 — [[knowledge/devops/devops知识体系/11-AIOps智能运维\|📄]]

| 模块 | 要点 |
|:----|:------|
| **AI 代码评审** | MR/PR 自动审查、语义级规范检查、逻辑缺陷检测、自研架构 |
| **智能构建诊断** | 错误日志分类、根因分析（RCA）、修复建议生成、历史故障匹配 |
| **流水线智能优化** | 测试影响分析（TIA）、构建加速、资源调度优化、失败预测 |
| **大模型工程化** | API 调用 vs 私有化部署（vLLM/Ollama）、RAG 增强、数据安全 |
| **AIOps 落地路线** | Phase 1 代码评审 → Phase 2 构建诊断 → Phase 3 流水线优化 |

## 十二、多技术栈构建与专项工具 🆕 — [[knowledge/devops/devops知识体系/12-多技术栈构建与专项工具\|📄]]

| 模块 | 要点 |
|:----|:------|
| **Repo 多仓库管理** | manifest 清单、repo sync/start/upload、AOSP 式大规模项目管理 |
| **Gerrit 代码评审** | 单 Commit 评审、+1/+2 权限模型、Jenkins Gerrit Trigger 集成 |
| **KubeSphere** | 多租户、可视化流水线、灰度发布、内置可观测性 |
| **Java 构建** | Maven (pom.xml/GAV/生命周期)、Gradle (Task/增量构建/Wrapper) |
| **C++ 依赖管理** | Conan (conanfile.py)、vcpkg (Manifest 模式)、二进制缓存 |
| **效能度量** | DORA 四大指标、数据采集、Grafana 效能看板 |
