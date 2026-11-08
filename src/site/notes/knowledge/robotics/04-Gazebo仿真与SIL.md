---
{"dg-publish":true,"permalink":"/knowledge/robotics/04-Gazebo仿真与SIL/","title":"Gazebo 仿真与 SIL 闭环","tags":["ROS2","Gazebo","Fortress","SIL","仿真回归","CI门禁","机器人"],"dg-note-properties":{"title":"Gazebo 仿真与 SIL 闭环","date":"2026-11-07","tags":["ROS2","Gazebo","Fortress","SIL","仿真回归","CI门禁","机器人"]}}
---


# # 🎮 ROS 2 阶段三实战：Gazebo 仿真与 SIL 闭环

## 摘要

这是整个学习清单里**最值钱也最花时间**的一环 —— 越疆 JD 明确写着「SIL/HIL 实际搭建或使用经验」，这是硬门槛。

本文完成了一个**可判通过/失败的结构化 SIL 门禁**：启动 Gazebo → spawn 差速车 → 桥接 ROS 2 → 闭环控制 → 断言到达精度与耗时 → 输出 JSON → 退出码驱动 CI。

**实测结果**：目标 1.0m，实际 1.0123m（误差 1.2%），耗时 3.48s，`passed: true`。失败路径（超时）也验证过，退出码正确为 1。

**最大的坑**：headless 环境下 `Sensors` 系统插件会导致整个 Gazebo 服务端卡死 —— 这个问题会浪费几个小时，第 6 章有完整的定位过程。

> ⚠️ **重要后续**：本文 §6 当时把根因归为「headless 不需要渲染」，结论是禁用传感器。**2026-11-08 装完 NVIDIA 驱动后证明这个判断不完整** —— 真正原因是机器缺 GPU 驱动，`ogre2` 拿不到渲染上下文。**有显卡的机器装好驱动后，Sensors 插件完全正常，激光雷达和相机都能跑**（见 [[knowledge/robotics/01-环境搭建\|环境搭建（含 GPU 附录）]]）。§6 保留作排查方法论，但结论已被修正。

---

## 一、整体架构

```
┌─────────────┐   ign topics    ┌──────────────┐   ROS 2 topics   ┌────────────┐
│   Gazebo    │ ──────────────> │ ros_gz_bridge │ ───────────────> │ sil_test   │
│  (physics)  │ <────────────── │               │ <─────────────── │ (driver)   │
└─────────────┘   /cmd_vel      └──────────────┘   /cmd_vel        └────────────┘
       │                                                │
       │ /odom                                          │ 断言
       └────────────────────────────────────────────────┘
```

**关键理解**：Gazebo 和 ROS 2 是两套独立的通信系统（Ignition Transport vs DDS）。`ros_gz_bridge` 是唯一桥梁。**桥没配好，两边就是瞎子。**

---

## 二、世界文件

```xml
<physics name="1ms" type="ignored">
  <max_step_size>0.001</max_step_size>   <!-- 1ms 步长，影响精度和性能 -->
  <real_time_factor>1.0</real_time_factor>
</physics>

<!-- 必须的三个系统插件 -->
<plugin filename="ignition-gazebo-physics-system" name="...Physics"/>
<plugin filename="ignition-gazebo-user-commands-system" name="...UserCommands"/>
<plugin filename="ignition-gazebo-scene-broadcaster-system" name="...SceneBroadcaster"/>
```

**三个插件缺一不可**：

| 插件 | 作用 | 缺了会怎样 |
|---|---|---|
| Physics | 物理引擎 | 世界不动 |
| UserCommands | 处理 spawn/删除请求 | `ros_gz_sim create` 超时 |
| SceneBroadcaster | 发布场景状态 | `/world/default/pose/info` 没有 |

再加上障碍物（一个 box 一个 cylinder），为后续导航场景准备。

---

## 三、机器人模型（SDF）

### 3.1 与 URDF 的差异

| | URDF | SDF |
|---|---|---|
| 用途 | ROS 2 描述（TF/可视化） | Gazebo 仿真（含物理参数） |
| 传感器 | 不支持 | 原生支持 |
| 关节限位 | `<limit>` | `<limit>` + 摩擦/阻尼 |
| 插件 | 不支持 | `<plugin>` 挂系统 |

**实践中的组合**：URDF 描述机器人结构（给 RViz 和导航用），SDF 描述仿真行为（给 Gazebo 用）。两者要保持一致，否则「RViz 里是 A 车，仿真里是 B 车」。

### 3.2 差速驱动插件

```xml
<plugin filename="ignition-gazebo-diff-drive-system"
        name="ignition::gazebo::systems::DiffDrive">
  <left_joint>wheel_left_joint</left_joint>
  <right_joint>wheel_right_joint</right_joint>
  <wheel_separation>0.32</wheel_separation>   <!-- 两轮间距 -->
  <wheel_radius>0.06</wheel_radius>
  <topic>cmd_vel</topic>                       <!-- 接收速度指令 -->
  <odom_topic>odom</odom_topic>                <!-- 发布里程计 -->
  <odom_publish_frequency>20</odom_publish_frequency>
</plugin>
```

**`wheel_separation` 必须与模型实际轮距一致**，否则转向半径算错，odom 会有累积偏差。这是仿真与真机对不上的常见原因。

### 3.3 传感器（本文档中禁用，见第六章）

```xml
<sensor name="lidar" type="gpu_lidar">
  <topic>scan</topic>
  <gz_frame_id>laser</gz_frame_id>   <!-- 关键：TF 帧名 -->
  <lidar>
    <scan><horizontal>
      <samples>360</samples>
      <min_angle>-3.14159</min_angle>
      <max_angle>3.14159</max_angle>
    </horizontal></scan>
    <range><min>0.12</min><max>12.0</max></range>
  </lidar>
</sensor>
```

**`gz_frame_id` 不能漏**：它决定 LaserScan 消息里的 `frame_id`，漏了后 TF 查询会失败，SLAM 直接崩。

---

## 四、ros_gz_bridge 配置

```bash
ros2 run ros_gz_bridge parameter_bridge \
  /cmd_vel@geometry_msgs/msg/Twist@ignition.msgs.Twist \
  /odom@nav_msgs/msg/Odometry@ignition.msgs.Odometry \
  /clock@rosgraph_msgs/msg/Clock@ignition.msgs.Clock \
  /joint_states@sensor_msgs/msg/JointState@ignition.msgs.JointState \
  /tf@tf2_msgs/msg/TFMessage@ignition.msgs.Pose_V
```

### 语法说明

```
/topic@<ROS类型>@<Ignition类型>
```

### 方向符号

| 符号 | 方向 | 例子 |
|---|---|---|
| `@` | 双向 | `/cmd_vel`（发指令也要收状态） |
| `[` | Ignition → ROS | `/odom`（仿真发，ROS 收） |
| `]` | ROS → Ignition | `/cmd_vel`（ROS 发，仿真收） |

**实测用的全是双向 `@`**，够用且不会出错。CI 里为了严谨可以显式指定方向，避免意外回环。

### `/clock` 为什么必须桥

ROS 2 里 `use_sim_time:=true` 的节点依赖 `/clock` 才知道「仿真时间」。**不桥 `/clock`，所有定时器都会用墙上时间**，仿真跑快跑慢都和实际不符，测试超时判断全乱。

---

## 五、SIL 测试用例实现

### 5.1 设计思路

测试用例不是「点一下看看动不动」，而是**可判定的门禁**：

```python
self.declare_parameter('target_distance', 1.5)
self.declare_parameter('tolerance', 0.15)
self.declare_parameter('max_duration_sec', 15.0)
```

三个维度同时断言：

| 维度 | 判据 | 意义 |
|---|---|---|
| **是否到达** | `status == REACHED` | 功能正确性 |
| **精度** | `error <= tolerance` | 控制质量 |
| **性能** | `duration <= max_duration` | 不能靠慢速蒙混过关 |

### 5.2 结构化输出

```json
{
  "status": "REACHED",
  "passed": true,
  "target_distance": 1.0,
  "actual_distance": 1.0123,
  "error_m": 0.0123,
  "tolerance": 0.15,
  "duration_sec": 3.48,
  "max_duration_sec": 15.0,
  "linear_speed": 0.3
}
```

**为什么要 JSON 而不是纯文本**：

1. 看板可以直接解析（Grafana / 自研平台）
2. 历史趋势可对比（这次 1.0123m，下次 1.0210m 是不是退化？）
3. CI 可以从字段里提取做二次判断

### 5.3 退出码驱动

```python
sys.exit(0 if ok else 1)
```

**这是 SIL 门禁与「跑个 demo 看看」的本质区别**：CI 只认退出码。

### 5.4 实测结果

**通过路径**：

```
===SIL_RESULT===
{"status": "REACHED", "passed": true, "target_distance": 1.0,
 "actual_distance": 1.0123, "error_m": 0.0123, "duration_sec": 3.48}
===END===
exit=0
```

**失败路径**（目标 5m 但只给 6 秒，物理上不可能到达）：

```
{"status": "TIMEOUT", "passed": false, "target_distance": 5.0,
 "actual_distance": 1.4524, "error_m": 3.5476, "duration_sec": 6.04}
[ros2run]: Process exited with failure 1
exit_code=1
```

**两条路径都验证了，门禁是真的在判，不是摆设。**

---

## 六、最大的坑：headless 下 Sensors 插件卡死

### 6.1 现象

在 `robot_world.sdf` 里加了 Sensors 系统插件后：

```xml
<plugin filename="ignition-gazebo-sensors-system" name="...Sensors">
  <render_engine>ogre2</render_engine>
</plugin>
```

- Gazebo 服务端能启动，日志显示世界已加载
- 但 `/gazebo/worlds` 服务调用**永久超时**
- `ros_gz_sim create` 也超时
- **整个仿真变成僵尸状态，不崩不响应**

### 6.2 定位过程

**第一步：确认进程还在**

```bash
pgrep -c -f 'ign gazebo'    # 3 个进程，说明没崩
```

**第二步：看栈回溯**（关键线索）

```
#15  libignition-gazebo-sensors-system.so  ...SensorsPrivate::RenderThread()
#14  libignition-gazebo-sensors-system.so  ...SensorsPrivate::WaitForInit()
#13  libignition-gazebo6-rendering.so.6    ...RenderUtil::Init()
#12  libignition-rendering6.so.6           ...RenderEngineManager::Engine()
```

**卡在 `WaitForInit`** —— 渲染引擎初始化永远不返回。

**第三步：确认渲染库存在**（排除「没装」）

```bash
ls /usr/lib/x86_64-linux-gnu/ign-rendering-6/engine-plugins/
# libignition-rendering6-ogre2.so  ← 在，不是缺库
```

**第四步：验证无 Sensors 插件时是否正常**

```bash
# 注释掉 Sensors 插件后重启
ign model --list    # → 立即返回，模型列表正常
ros_gz_sim create   # → OK creation of entity
```

**结论：问题在 Sensors 插件本身，不在 Gazebo 或模型。**

### 6.3 根因

`ogre2` 渲染引擎在**无 GPU 上下文 / 无 EGL 环境**时会**阻塞等待**，而不是报错退出。`--headless-rendering` 只是「不显示窗口」，**不代表不需要渲染能力**。

**试过但没用的方案**：

```bash
export LIBGL_ALWAYS_SOFTWARE=1
export GALLIUM_DRIVER=llvmpipe
# → 软件渲染更慢，初始化一样卡住
```

### 6.4 正确做法

**方案 A（无 GPU 时的降级方案，非首选）：禁用传感器，只跑物理 + 里程计**

> 划线更正：这是**没有 GPU 驱动时**的临时方案，不是 CI 的推荐做法。有显卡的机器装好驱动后应走方案 B。


```xml
<!-- 无 GPU 环境注释掉 -->
<!--
<plugin filename="ignition-gazebo-sensors-system" name="...Sensors">
  <render_engine>ogre2</render_engine>
</plugin>
-->
```

**为什么这样就够**：

- SIL 的核心是验证**控制逻辑与集成链路**，不需要真实图像
- 里程计足以断言「到没到、误差多少、耗时多少」
- 没有渲染，仿真反而跑得更快更稳

**方案 B：真需要传感器时**

| 环境 | 做法 |
|---|---|
| 有 GPU 的 CI runner | 挂 `--gpus all`，或裸机跑 |
| 无 GPU 服务器 | 用 `xvfb-run` + `vglrun`，或接受性能损失 |
| 只测激光雷达 | 用 `ray`（CPU）不用 `gpu_ray` |

> **`ray` vs `gpu_ray`**：`ray` 用 CPU 做射线检测，慢但不需要渲染；`gpu_ray` 依赖 GPU 渲染管线。**CI 里优先 `ray`**，这是本文档之外的另一条实用经验。

### 6.5 教训

> **CI 里跑仿真，「能跑」和「能稳定跑」是两件事。** 任何依赖 GPU 的组件都要在 CI 环境里先行验证，否则会在改完代码后才发现「本地能跑，CI 挂了」。

**补充（2026-11-08）**：本次的「卡死」最终定位到**机器没装 GPU 驱动**，不是 headless 的固有限制。装好 NVIDIA 595 驱动后，同一份配置在 headless 下能正常出 `/scan` 和 `/camera/image_raw`。

所以真正的教训要再进一步：

> **看到「渲染相关组件静默卡死」，第一反应应该是查 `nvidia-smi`，而不是改配置绕过。** 用 `nouveau` 开源驱动时 `lsmod` 看起来一切正常、`lspci` 也能看到显卡，唯独 `ogre2` 拿不到渲染上下文——这种「硬件在、驱动假在」的状态最容易误判。

这也正好是一个**机器人 CI 区别于普通 CI 的独有门禁项**：环境能力探测（是否有 GPU、是否支持 EGL）。

---

## 七、其他坑

### 7.1 参数类型必须匹配

```bash
ros2 run hello_robot square_driver --ros-args -p timeout_sec:=20
# → InvalidParameterTypeException: Trying to set parameter 'timeout_sec'
#    to '20' of type 'INTEGER', expecting type 'DOUBLE'
```

**`20` 被解析成 INTEGER，`20.0` 才是 DOUBLE。** 声明为 double 的参数必须传浮点字面量。

**CI 里的建议**：所有参数统一用 double，launch 文件里写 `'20.0'`，避免这种低级错误浪费构建时间。

### 7.2 改了 setup.py 必须干净重编

```bash
colcon build --packages-select hello_robot   # ❌ 入口点没生效
rm -rf build/hello_robot install/hello_robot
colcon build --packages-select hello_robot   # ✅
```

**症状**：

```
StopIteration   # 或者 pkg_resources DistributionNotFound
```

新增 `console_scripts` 入口点时，setuptools 的 egg-info 缓存不会自动更新。**`--cmake-clean-cache` 对 Python 包无效**，必须删目录。

### 7.3 `setsid`/`nohup` 起 Gazebo 会丢日志

```bash
setsid ign gazebo ... > /tmp/gz.log 2>&1 &
# → /tmp/gz.log 文件根本不存在
```

**本次用 OpenClaw 的持久后台会话解决**（进程组不脱离控制）。手动操作时用 `tmux new -d -s gz 'ign gazebo ...'`。

---

## 八、CI 集成方案

### 8.1 流水线骨架

```yaml
# .gitlab-ci.yml 或 Jenkinsfile 等价结构
sil_regression:
  stage: test
  image: ros:humble
  script:
    # 1. 启动仿真（后台）
    - ign gazebo -s -r --headless-rendering $WORLD &
    - sleep 10
    # 2. spawn 机器人
    - ros2 run ros_gz_sim create -name diffbot -file $MODEL
    # 3. 启动桥接
    - ros2 run ros_gz_bridge parameter_bridge ... &
    - sleep 5
    # 4. 跑用例（退出码即结果）
    - ros2 run hello_robot sil_test --ros-args -p target_distance:=1.0
  artifacts:
    when: always
    paths:
      - sil_result.json
      - "*.log"
```

### 8.2 必须做的三件事

| 项 | 为什么 |
|---|---|
| **失败也归档日志**（`when: always`） | 失败时的日志才最有价值 |
| **保存 rosbag** | 事后回放比看日志快十倍 |
| **固定超时** | 否则卡死的仿真会占满 runner |

### 8.3 量化门禁建议阈值

```python
tolerance = 0.15          # 15cm，差速车仿真合理范围
max_duration = target / speed * 1.5   # 理论耗时 × 1.5
# 例：1.0m / 0.3m/s × 1.5 = 5s
```

---

## 九、下一步

阶段 3 的核心目标（SIL 闭环）已达成。后续可选深化：

1. **加激光雷达场景测试**（用 `ray` 不用 `gpu_ray`）→ 障碍检测用例
2. **多场景参数化**（正常/撞障/超时三类用例）
3. **接入 SLAM + Nav2** → 导航到点的端到端测试（阶段 4）
4. **rosbag 自动录制** → 失败现场保全

---

## 相关文档

- [[knowledge/robotics/01-环境搭建\|01-环境搭建]] — 环境基线（含 headless 基础验证）
- [[knowledge/robotics/02-节点通信与TF2\|02-节点通信与TF2]] — 节点与 URDF
- [[knowledge/robotics/03-colcon构建体系\|03-colcon构建体系]] — 构建与增量
- [00-机器人入门路线](00-机器人入门路线.md) — 路线规划