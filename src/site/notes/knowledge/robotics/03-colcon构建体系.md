---
{"dg-publish":true,"permalink":"/knowledge/robotics/03-colcon构建体系/","title":"colcon 构建体系深挖","tags":["ROS2","colcon","ament","构建系统","依赖图","overlay","CI"],"dg-note-properties":{"title":"colcon 构建体系深挖","date":"2026-11-07","tags":["ROS2","colcon","ament","构建系统","依赖图","overlay","CI"]}}
---


# # 🔧 ROS 2 阶段二实战：colcon 构建体系深挖

## 摘要

colcon 是机器人 CI 岗的**分水岭技能**。本文用五个可复现实验把它的行为摸清楚：依赖 DAG 解析、增量构建的传播边界、`--packages-*` 选择器的实际含义、overlay 叠加规则、`--merge-install` 的取舍。

**最重要的发现**：改接口消息后，下游包的重编时间反而比改实现更短（1.14s vs 3.16s）——因为接口变更触发的是「重新生成 + 重新链接」，而实现变更触发的是「重新编译」。**这直接决定了 CI 里该怎么切缓存**。

---

## 一、实验工作区

```
~/ros2_ws/src/
├── hello_robot/       # ament_python（Python 节点）
├── robot_interfaces/  # ament_cmake（纯消息包，被依赖）
└── robot_core/        # ament_cmake（C++ 节点，依赖 robot_interfaces）
```

依赖关系（由 `colcon graph --dot` 实测）：

```
"robot_core" -> "robot_interfaces"
"hello_robot"   （独立，无依赖）
```

---

## 二、实验一：全新构建

```bash
rm -rf build install log
colcon build
```

**实测输出**：

```
Starting >>> hello_robot                ← 与 interfaces 并行（无依赖）
Finished <<< hello_robot [0.53s]
Finished <<< robot_interfaces [2.03s]
Starting >>> robot_core                 ← 必须等 interfaces 完成
Finished <<< robot_core [3.53s]
Summary: 3 packages finished [5.73s]
TOTAL=5.86 s
```

**关键观察**：

- colcon **按依赖拓扑分层并行**，不是串行也不全并行
- `hello_robot` 和 `robot_interfaces` 同层 → 同时启动
- `robot_core` 的 `Starting` 出现在 interfaces 的 `Finished` 之后 → 依赖门控生效

---

## 三、实验二：增量构建的传播边界

### 实验 2a：无改动

```bash
colcon build
# Summary: 3 packages finished [0.70s]   T1=0.82 s
```

0.7 秒做完三个包——colcon 检测到无变更，**连编译都没启动**，只是走一遍检查流程。

### 实验 2b：只改 C++ 实现

```bash
touch src/robot_core/src/core_node.cpp
colcon build
```

```
Starting >>> robot_interfaces          ← 也会启动，但几乎瞬间完成
Starting >>> hello_robot
Finished <<< robot_interfaces [0.26s]
Starting >>> robot_core
Finished <<< robot_core [2.74s]          ← 真正的重编在这里
Summary: 3 packages finished [3.16s]
T2=3.28 s
```

**只有 `robot_core` 真的重编了**，`robot_interfaces` 那 0.26 秒是 CMake 配置检查（发现无变更立即退出）。

### 实验 2c：改接口消息定义

```bash
echo "# 注释改动" >> src/robot_interfaces/msg/RobotStatus.msg
colcon build
```

```
Finished <<< robot_interfaces [0.87s]    ← 重新生成消息代码
Starting >>> robot_core
Finished <<< robot_core [0.10s]          ← 只是重新链接，没重编
Summary: 3 packages finished [1.14s]
T3=1.27 s
```

### 三组数据对比

| 场景 | 耗时 | 谁被重编 |
|---|---|---|
| 无改动 | 0.82 s | 无 |
| 改 C++ 实现 | 3.28 s | robot_core |
| 改接口定义 | 1.27 s | robot_interfaces 重新生成 + robot_core 重新链接 |

**这就是「增量构建的传播边界」**：

- 改**叶子包实现** → 只影响自己
- 改**被依赖包的头文件/接口** → 影响整条下游链（本次只有一个下游，规模化项目里可能是几十个）

**对 CI 的直接意义**：

> 大规模机器人工程里，「改一个 msg」可能导致几十个包重编。CI 必须按**影响面**切分构建：改接口跑全量下游，改实现只跑自己 + 直接消费者。这正是车载 CI 里「部件粒度增量构建」在 ROS 2 里的对应实现。

---

## 四、实验三：`--packages-*` 选择器的真实语义

这四个参数长得像，语义完全不同，**面试高频考点**：

```bash
colcon list --packages-select robot_core
# → robot_core                    只它自己

colcon list --packages-up-to robot_core
# → robot_core, robot_interfaces  它 + 它的所有依赖

colcon list --packages-above robot_interfaces
# → robot_interfaces, robot_core  它 + 所有依赖它的（下游）

colcon list --packages-above-and-dependencies robot_interfaces
# → robot_interfaces, robot_core  完整闭包（本次上下游重合）
```

**记忆口诀**：

| 参数 | 方向 | 用途 |
|---|---|---|
| `select` | 只自己 | 单包验证 |
| `up-to` | 向上（找依赖） | 首次构建某包 |
| `above` | 向下（找消费者） | **改接口后跑影响面** |
| `above-and-dependencies` | 双向闭包 | 全链路验证 |

**为什么 `above` 最值钱**：PR 改了 `robot_interfaces`，CI 只需要跑 `--packages-above robot_interfaces`，不用全量。这就是「受影响的部件 + 依赖面」在 ROS 2 的实现。

---

## 五、实验四：overlay 叠加机制

### 步骤

```bash
# 1. 复制一个包到新工作区
mkdir -p ~/overlay_ws/src
cp -r ~/ros2_ws/src/hello_robot ~/overlay_ws/src/

# 2. 改版本号，便于区分
cd ~/overlay_ws
sed -i "s/version='0.1.0'/version='0.2.0-overlay'/" src/hello_robot/setup.py
colcon build --packages-select hello_robot

# 3. 只 source overlay
source install/setup.bash
ros2 pkg prefix hello_robot
```

### 实测结果

```
# source underlay 时：
/home/yang/ros2_ws/install/hello_robot

# source overlay 后：
/home/yang/overlay_ws/install/hello_robot      ← 覆盖成功

# AMENT_PREFIX_PATH 顺序：
/home/yang/overlay_ws/install/hello_robot       ← 排第一
/home/yang/ros2_ws/install/robot_core
/home/yang/ros2_ws/install/robot_interfaces
```

### 机制说明

`source overlay/install/setup.bash` 会**自动 source underlay**（因为构建时 `--merge-install` 记录了 underlay 路径），然后把 overlay 前缀**插到 `AMENT_PREFIX_PATH` 最前面**。

**查找规则是按路径顺序取第一个匹配**，所以 overlay 里的同名包会遮蔽 underlay 的。

### 「明明编了却没生效」的三种成因

| 成因 | 判断方法 |
|---|---|
| 没 source 新工作区 | `echo $AMENT_PREFIX_PATH` 看顺序 |
| 顺序错了（老工作区在前） | 路径顺序错乱 → 重新按序 source |
| 同名包在多个工作区 | `ros2 pkg prefix <name>` 看实际解析到哪 |

**CI 里的建议**：镜像里只 source 一个工作区，不要叠加多个。overlay 是开发期便利，部署期是风险。

---

## 六、实验五：`--merge-install` vs 隔离安装

### 隔离安装（默认）

```
install/
├── hello_robot/
│   ├── lib/
│   └── share/
├── robot_core/
├── robot_interfaces/
└── setup.bash
```

每个包有独立前缀，`ros2 pkg prefix robot_core` → `install/robot_core`。

### `--merge-install`

```
install/
├── include/     ← 所有包头文件合并
├── lib/         ← 所有 .so 和可执行文件合并
│   ├── hello_robot/
│   ├── librobot_interfaces__rosidl_generator_c.so
│   ├── librobot_interfaces__rosidl_generator_py.so
│   ├── librobot_interfaces__rosidl_typesupport_cpp.so
│   └── ...
└── setup.bash
```

所有包共享一个前缀。

### 对比

| 维度 | 隔离安装 | `--merge-install` |
|---|---|---|
| 目录层级 | 深（每包一层） | 扁平 |
| 单包定位 | 精确 | 需查索引 |
| 同名文件冲突 | 天然隔离 | **会互相覆盖** |
| 适合场景 | 日常开发 | **CI 镜像 / 部署** |
| `AMENT_PREFIX_PATH` 长度 | 包数 × 1 | 恒为 1 |

**CI 用 `--merge-install` 的理由**：

1. `AMENT_PREFIX_PATH` 短 → 环境变量不超长（包多了会撞 Linux 的 env 限制）
2. 镜像层更少 → 体积小、拉取快
3. 运行时查找快 → 路径解析少跳转

**风险**：同名可执行文件会互相覆盖。大工程要配合包名前缀规范。

---

## 七、坑点与排错

### 7.1 CMake 缓存污染

改了 `CMakeLists.txt` 的 `find_package` 后，colcon 可能仍用旧缓存：

```bash
colcon build --cmake-clean-cache
# 或只清某个包：
colcon build --packages-select robot_core --cmake-clean-cache
```

### 7.2 部分失败时继续构建

```bash
colcon build --continue-on-error -j 8
```

默认一个包失败会**中止整轮**。CI 里想跑完所有包拿到完整报告，加这个参数。

### 7.3 构建日志在哪

```bash
log/latest_build/<pkg>/stdout_stderr.log    # 编译输出
log/latest_build/<pkg>/command.log          # 实际执行的命令
log/latest_build/events.log                 # 全部事件（含依赖调度）
```

**CI 排障从 `command.log` 开始**——它记录了 colcon 实际调用的完整命令，比自己拼 CMake 参数快得多。

### 7.4 依赖没装

```bash
rosdep install --from-paths src --ignore-src -r -y
# -r  未解析的依赖警告但不中止
# -y  自动确认
```

> 注意 rosdep 的 `index-v4.yaml` 需要 `ROSDISTRO_INDEX_URL` 镜像，见 [[knowledge/robotics/01-环境搭建\|01-环境搭建]] §2.4。

---

## 八、给 CI 的设计结论

这次实验直接产出四条 CI 设计原则：

### 1. 按影响面切构建

```bash
# 改了 robot_interfaces → 只跑影响面，不跑全量
colcon build --packages-above robot_interfaces
colcon test --packages-above robot_interfaces
```

### 2. 接口包变更要跑全下游测试

改 msg/srv 定义 → 下游包的**行为**可能变（字段增删、类型变更），必须跑它们的测试而不只是编译。

### 3. CI 镜像用 `--merge-install`

体积和环境变量都更友好。但要在流水线里保证包名不冲突。

### 4. 缓存策略按「接口 vs 实现」分两档

| 变更类型 | 缓存键 | 说明 |
|---|---|---|
| 改实现 | 包名 + 源码 hash | 只失效该包 |
| 改接口 | 接口包 hash | **下游全部失效** |

> 这是 ccache + Docker layer 缓存之外，ROS 2 特有的第二层缓存逻辑。这也是「车载部件粒度增量构建」迁移到机器人的核心认知。

---

## 九、下一步

阶段 2 完成后进入阶段 3（Gazebo 仿真与 SIL 闭环），这是越疆/它石 JD 里的硬门槛：

1. 差速车进 Gazebo 空世界
2. 加激光雷达 + 相机传感器插件
3. 键盘/程序控制 → 里程计
4. **headless 跑起来**（CI 前提）
5. pytest 用例：启动 → 下发 cmd_vel → 断言位置

---

## 相关文档

- [[knowledge/robotics/01-环境搭建\|01-环境搭建]] — 环境基线
- [[knowledge/robotics/02-节点通信与TF2\|02-节点通信与TF2]] — 节点与 URDF
- [[knowledge/devops/practices/06-构建系统实战-CMake与Bazel\|../devops/practices/06-构建系统实战-CMake与Bazel]] — CMake 底层原理
- [00-机器人入门路线](00-机器人入门路线.md) — 路线规划
