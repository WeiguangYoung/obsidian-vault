---
{"dg-publish":true,"permalink":"/knowledge/devops/ROS2具身智能CI实战/","tags":["ROS2","具身智能","CI/CD","机器人","Docker","Kubernetes","HIL","训练计划","求职"],"dg-note-properties":{"date":"2026-09-29","tags":["ROS2","具身智能","CI/CD","机器人","Docker","Kubernetes","HIL","训练计划","求职"]}}
---


# ROS2 + 具身智能 CI 构建工程师：8 周专项训练计划

> 目标岗位：**ROS2 + 具身智能 CI 构建工程师**
> 核心思路：**每周产出可写进简历的作品，不要只看书。**
> 最终能力链：代码提交 → 容器化编译 → 仿真回归 → 模型评测 → 镜像打包 → K8s 边缘部署 → 真机冒烟测试

---

## 前置要求 / 硬件清单

| 项 | 最低配置 | 说明 |
|---|---|---|
| 开发机 | 16G 内存 + 512G SSD + 4 核 | 同时跑 Gazebo + 编译 ROS2 工作空间，8G 机器会频繁 OOM |
| 仿真 GPU | **可选** | 仅 Isaac Sim 需要 RTX 显卡（≥8G 显存）；Gazebo 纯 CPU 可跑 |
| 目标板（第 8 周） | Jetson Orin Nano / 树莓派 4B / 任意 x86 工控机 | 能跑 Docker 即可，不必高性能 |
| 集群（第 7 周） | 本地单机 K3s / Kind | 单节点足够，不必多机 |

**时间预算**：工作日 ~1.5h/天 + 周末 ~4h/天 ≈ **15–18 小时/周**。全职投入可压缩到 4–5 周，业余则需拉长，别按「8 周 = 两个月假期」估。

---

## 总览

| 周次 | 主题 | 交付仓库 |
|---|---|---|
| 1 | Linux + Python/C++ + Git + CMake | `ros2_basic_nodes` |
| 2 | ROS2 通信与调试 | `ros2_sensor_control_demo` |
| 3 | Docker 化 ROS2 环境 | `ros2-docker-template` |
| 4 | CI/CD 流水线 | `.github/workflows/ci.yml` |
| 5 | 仿真回归测试 | `ros2-simulation-regression` |
| 6 | 模型评测流水线 | `embodied-model-eval` |
| 7 | K8s 边缘部署 | `ros2-k8s-edge-deploy` |
| 8 | HIL / 真机冒烟测试 | `ros2-hil-pipeline` |

---

## 第 1 周：Linux + Python/C++ + Git + CMake

**目标**：能独立编译、调试、管理一个 ROS2 包。

### 练什么

- Ubuntu 22.04 或 24.04 装好 ROS2 Humble / Jazzy。
- 写一个 Python 节点和一个 C++ 节点，互相发 Topic。
- 用 `CMakeLists.txt` / `setup.py` 正常编译。
- 用 Git 管理代码，提交信息规范。

> ⚠️ **一周装满 8 项偏满**：零基础者实为 2 周的量。若已有 CMake/Git 基础 → 1 周可行；否则第 1 周只做 **Python 节点 + Git 规范**，C++ 节点与 CMake 挪到第 2 周（Topic 互通正好用得上）。

### 交付物

- 一个 GitHub 仓库：`ros2_basic_nodes`
- 包含：Python 节点、C++ 节点、CMake 构建、README。

---

## 第 2 周：ROS2 通信与调试

**目标**：熟悉 Node、Topic、Service、Action、Parameter、Launch。

### 练什么

- 写一个传感器模拟节点，发布 `/imu`、`/camera_info`。
- 写一个控制节点，订阅数据并发布 `/cmd_vel`。
- 用 `ros2 launch` 启动多个节点。
- 用 `ros2 topic echo/hz`、`ros2 service call`、`rviz2` 调试。
- 用 `rosbag2` 录制和回放数据。

### 交付物

- 仓库：`ros2_sensor_control_demo`
- 能一键 launch 启动，并录制 rosbag。

---

## 第 3 周：Docker 化 ROS2 环境

**目标**：把 ROS2 项目放进容器，做到"换机器也能跑"。

### 练什么

- 写 Dockerfile，基于 `ros:humble` 或 `ros:jazzy`。
- 容器里编译 colcon 工作空间。
- 用 `docker-compose` 启动多个节点。
- 挂载 rosbag、rviz2 配置。
- 处理 GPU/CUDA 或仿真依赖时，先做 CPU 版本，再逐步加 GPU。

### 交付物

- 仓库：`ros2-docker-template`
- 命令：`docker compose up` 即可跑通仿真或节点。

---

## 第 4 周：CI/CD 流水线

**目标**：提交代码后自动编译、测试、打包镜像。

### 练什么

- 用 GitHub Actions 或 GitLab CI。
- 流水线步骤：checkout → 安装依赖 → `colcon build` → 单元测试 → 打包镜像。
- 缓存 ROS 依赖和 pip/apt 包，避免每次全量重装。
  - **落地要点**：`apt` 依赖**能进自定义基础镜像的就进镜像**（`ros:humble` 基础上预装常用依赖并推私有 registry），CI 只装增量 —— 比在 workflow 里逐包 apt 更快更稳。
  - ROS 2 不用逐包 apt 装，**直接以 `ros:humble` 官方镜像为 job 容器**。
  - `pip` 用 `actions/cache` 缓存 `~/.cache/pip`；colcon 工作空间可用 `ccache` 加速重编。
- 失败时输出日志、构建产物和测试报告。

### 交付物

- `.github/workflows/ci.yml`
- 每次 push 触发：
  - 编译通过
  - 单元测试通过
  - 生成 Docker 镜像

---

## 第 5 周：仿真回归测试

**目标**：代码改动后自动跑仿真，判断是否退化。

### 练什么

- 用 **Gazebo（Classic / Ignition）** 跑一个小车/机械臂场景。
  - ⚠️ **别在 CI 里用 Isaac Sim**：它需要 RTX GPU + 显存 ≥8G，无头模式同样吃 GPU，GitHub Actions 这类托管 runner 跑不了。CI 一律用 **Gazebo（纯 CPU 可行）**，Isaac Sim 只作为本地进阶选项。
- 写自动化用例：启动仿真 → 发送目标 → 检查是否到达/是否碰撞。
- 无头（headless）运行仿真，CI 里也能跑。
- 输出通过率、延迟、碰撞次数、任务完成时间。

### 交付物

- 仓库：`ros2-simulation-regression`
- CI 里能跑仿真用例，并生成报告。

### 量化门禁（在本周就定义，第 8 周直接复用）

| 指标 | 建议门禁 |
|---|---|
| 用例通过率 | `< 90%` → 阻断 |
| 任务完成时间 | 超基线 `> 20%` → 阻断 |
| 碰撞次数 | `> 0` → 阻断（安全类场景） |
| 端到端延迟 | 超阈值（如 `> 200ms`）→ 阻断 |

> 门禁要写成**数字**，不能只说「失败就阻断」——否则 CI 判不出通过与否。

---

## 第 6 周：模型评测流水线

**目标**：把具身智能模型接入 CI。

### 练什么

- 准备一个策略模型或感知模型，比如 ONNX / PyTorch。
- 写评测脚本：输入传感器数据，输出动作/检测结果。
- 指标：推理延迟、CPU/GPU 占用、成功率、错误率。
- 每次模型更新自动跑评测，生成 Markdown / JSON 报告。
  - **触发方式要明确**：最简可落地 = **模型文件入库 + path filter 触发**，即
    ```yaml
    on:
      push:
        paths: ['models/**', 'eval/**']
    ```
    避免「每次 push 都跑模型评测」（慢且浪费）；也不建议依赖外部模型仓库 webhook（复杂、难复现）。

### 交付物

- 仓库：`embodied-model-eval`
- 提交新模型后，CI 自动输出评测表。

---

## 第 7 周：K8s 边缘部署

**目标**：把 ROS2 节点和模型服务部署到集群或边缘节点。

### 练什么

- 把推理服务容器化，例如 FastAPI + ONNX Runtime。
- 写 Kubernetes Deployment、Service、ConfigMap。
- 用 Helm 或 Kustomize 管理多环境配置。
- 部署到本地 K3s / Kind 集群，模拟边缘节点。
- 接入 Prometheus + Grafana，监控延迟、重启次数、资源占用。

### 交付物

- 仓库：`ros2-k8s-edge-deploy`
- 一键部署推理服务 + ROS2 桥接节点。

---

## 第 8 周：HIL / 真机冒烟测试

**目标**：让 CI 不只跑仿真，也能触达真实硬件。

### 练什么

- 准备一台目标板：Jetson、工控机、树莓派或机械臂。
- CI 触发远程脚本，把镜像推送到目标板。
- 目标板自动启动节点，跑冒烟测试。
- 采集日志、指标、传感器数据并回传。
- 设置门禁：编译失败、仿真失败、真机超时都阻断合并（阈值复用第 5/6 周的量化门禁）。

> ⚠️ **网络与权限现实约束**（最容易卡住的一步）：
> - CI runner 必须**能访问到目标板**。托管 runner 到不了内网 → 用**自建 runner**（跑在能通目标板的机器上），或 **WG / frp 隧道**打通。
> - SSH **密钥走 CI Secret**，绝不入库；目标板侧建议专用 `ci` 用户 + 最小权限。
> - 冒烟测试要**幂等 + 可重跑**：先 `docker rm -f` 清场、固定端口、超时 `timeout 300` 兜底，避免残留进程污染下次构建。

### 交付物

- 仓库：`ros2-hil-pipeline`
- README 里写清楚：如何接入目标板、如何跑真机测试。

---

## 最终作品集建议

把这 8 周做成一个完整项目集：

1. `ros2_basic_nodes`：基础节点。
2. `ros2-docker-template`：容器化模板。
3. `ros2-simulation-regression`：仿真回归。
4. `embodied-model-eval`：模型评测。
5. `ros2-k8s-edge-deploy`：边缘部署。
6. `ros2-hil-pipeline`：HIL/真机测试。

**面试时重点讲这条链路**：

> 代码提交 → 容器化编译 → 仿真回归 → 模型评测 → 镜像打包 → K8s 边缘部署 → 真机冒烟测试

这比单独说"我会 ROS2"或"我会 CI"更有竞争力——因为它展示的是**端到端交付能力**，而不是孤立的工具使用。

---

## 执行要点（补遗）

- **硬件按前置清单备齐再开工**：16G 开发机是硬门槛，Gazebo 反复 OOM 会直接劝退。
- **Isaac Sim 不进 CI**：CI 全链路（第 4/5/6 周）用 Gazebo + CPU 推理即可跑通，GPU 只在本地做进阶验证。
- **门禁必须量化**：第 5/6 周就定数字阈值，第 8 周直接复用，别到最后一刻才纠结核不核。
- **模型评测触发用 path filter**：`on.push.paths: ['models/**']`，简单且可复现。
- **CI 到真机的连通性是第 8 周第一道坎**：自建 runner 或 WG/frp 隧道，密钥走 Secret。

---

## 备考提示

- **每周必须有一个可演示产物**：仓库 + README + 能跑的命令。没有产物的学习周等于浪费。
- **优先把链路串起来**：第 3 周起就要让 Docker 化产物进入第 4 周 CI，别等 8 周结束再拼。
- **仿真先 CPU 后 GPU**：CI 环境大多无 GPU，无头仿真 + CPU 推理能跑通第 4/5/6 周全部门禁，再迭代到 GPU。
- **第 7、8 周是差异化重点**：K8s 边缘部署 + HIL 真机门禁是多数 ROS2 候选人没有的能力，与 DevOps 背景天然契合。
