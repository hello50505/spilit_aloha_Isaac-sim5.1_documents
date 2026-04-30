# Isaac Sim ROS2 控制 Mobile ALOHA

本文记录当前已验证可用的 Mobile ALOHA ROS2 控制流程、关键文件、修改内容和修改原因。

当前目标：

- ROS2 通过 `/cmd_vel` 控制底盘真实位移。
- ROS2 通过 `sensor_msgs/JointState` 控制轮子视觉同步和机械臂测试动作。
- Isaac Sim 发布标准 `/joint_states`。
- 用户尽量只需要两个启动脚本，不再手动拼环境变量或在 Script Editor 里重复执行脚本。

下面命令默认在 `isaac-sim` 目录执行：

```bash
cd /home/xiangpc/dataset/isaac-sim
```

## 最终启动流程

### 1. 启动 Isaac Sim

宿主机终端执行：

```bash
./dev_ws/scripts/run_isaac_mobile_aloha.sh
```

这个脚本会：

- 清理 `PYTHONPATH`、`PYTHONHOME`、`LD_LIBRARY_PATH`、`CUDA_HOME`、`CUDA_PATH`、`CONDA_PREFIX`、`CONDA_DEFAULT_ENV`，避免 Conda 或系统 CUDA 污染 Isaac Sim standalone 自带 Torch/CUDA。
- 设置 `ROS_DISTRO=humble`。
- 设置 `RMW_IMPLEMENTATION=rmw_cyclonedds_cpp`，保持和 Docker 内 ROS2 一致。
- 只设置 Isaac ROS2 Bridge 需要的 `LD_LIBRARY_PATH` 和 `PYTHONPATH`。
- 使用 `--no-ros-env` 启动 Isaac Sim，避免宿主机 Conda/ROS 环境污染 Isaac。
- 使用 `--exec dev_ws/isaac/scripts/mobile_aloha_kinematic_base_controller.py` 自动挂载运动学底盘控制器。

Isaac Sim 打开后：

1. 打开 `assets/4.mobile_aloha_in_market.usda`。
2. 确认 `isaacsim.ros2.bridge` 已启用。
3. 点击 Play。

说明：`mobile_aloha_kinematic_base_controller.py` 支持先启动、后等待场景加载。即使它在 USD 打开前执行，也会等到机器人 prim 出现后再自动订阅 `/cmd_vel`。

### 2. 启动 ROS2 Docker 控制节点

另开宿主机终端：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
./scripts/run_ros2_humble_docker.sh --launch-control
```

这个脚本会：

- 使用 `mobile_aloha_ros2_humble:latest` 镜像。
- 新建或进入 `isaac_ros2_humble` 容器。
- 使用 `--net=host`，让 Docker、宿主机和 Isaac Sim 处在同一个 ROS2 网络。
- source `/opt/ros/humble`、Isaac Sim 官方 `humble_ws`、`dev_ws/ros2_ws/install/setup.bash`。
- 如果 `mobile_aloha_control.launch.py` 尚未运行，在当前终端前台启动它。
- 如果控制 launch 已经运行，再次执行脚本只进入容器 shell，不重复启动 `/cmd_vel` 订阅节点。

如果刚修改过 ROS2 包：

```bash
./scripts/run_ros2_humble_docker.sh --build --launch-control
```

如果还想同时启动机械臂 demo：

```bash
./scripts/run_ros2_humble_docker.sh --launch-control -- start_arm_demo:=true arm_side:=both
```

只进入容器、不启动控制节点：

```bash
./scripts/run_ros2_humble_docker.sh
```

### 3. 检查 ROS2 状态

进入容器 shell 后执行：

```bash
ros2 topic info /cmd_vel
ros2 topic info /isaac_joint_commands
ros2 topic info /isaac_joint_commands_interpolated
ros2 topic info /joint_states
```

正常状态应接近：

```text
/cmd_vel                Subscription count: 2
/isaac_joint_commands   Publisher count: 1+, Subscription count: 1
/isaac_joint_commands_interpolated   Publisher count: 1, Subscription count: 1
/joint_states           Publisher count: 1
```

`/cmd_vel` 的两个 subscriber 分别是：

- Isaac Sim 里的 `mobile_aloha_kinematic_base_controller.py`，负责真实移动整车根 Xform。
- Docker 里的 `cmd_vel_to_joint_command`，负责轮子转向/转动的视觉同步。

如果 `/cmd_vel` 只有 1 个 subscriber，通常说明 Isaac 侧运动学底盘控制器没运行，现象就是“轮子动但车身不动”。

## ROS2 话题接口

### 必用控制接口

| Topic | Type | 方向 | 用途 |
| --- | --- | --- | --- |
| `/cmd_vel` | `geometry_msgs/msg/Twist` | 外部功能发布 | 底盘速度命令。Isaac 侧运动学控制器用它移动整车根 Xform，Docker 侧 `cmd_vel_to_joint_command` 用它同步轮子转向和转动。 |
| `/isaac_joint_commands` | `sensor_msgs/msg/JointState` | 外部功能或本项目节点发布 | 关节目标入口。用于轮子视觉同步、机械臂测试动作，也可给后续上层功能直接控制指定关节。 |
| `/isaac_joint_commands_interpolated` | `sensor_msgs/msg/JointState` | `joint_command_interpolator` 发布，Isaac 订阅 | 内部平滑执行接口。上层不要直接发布这个 topic。 |
| `/joint_states` | `sensor_msgs/msg/JointState` | Isaac 以 200Hz 发布，外部功能订阅 | 机器人所有关节状态反馈，包括底盘轮、升降、双臂和夹爪关节。 |
| `/clock` | `rosgraph_msgs/msg/Clock` | Isaac 发布，外部功能订阅 | 仿真时间。需要按仿真时间运行的 ROS2 节点应设置 `use_sim_time:=true`。 |

### `/cmd_vel`

`/cmd_vel` 是底盘对外接口，适合遥控、导航、策略模型或任务脚本调用。

字段约定：

| 字段 | 含义 | 当前限制 |
| --- | --- | --- |
| `linear.x` | 车体前后速度，单位 m/s | `[-0.5, 0.5]` |
| `linear.y` | 车体左右平移速度，单位 m/s | `[-0.3, 0.3]` |
| `angular.z` | yaw 角速度，单位 rad/s | `[-1.0, 1.0]` |
| `linear.z`、`angular.x`、`angular.y` | 未使用 | 忽略 |

示例：

```bash
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.2, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.3}}"
```

停止：

```bash
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"
```

注意：当前 Isaac 运动学控制器有 `0.5s` 命令超时。持续运动时，上层功能应连续发布 `/cmd_vel`，不要只发一次。

### `/isaac_joint_commands`

`/isaac_joint_commands` 是关节目标入口，消息类型为 `sensor_msgs/msg/JointState`。只需要填要控制的关节名和对应命令，不必每次发送全部关节。

默认启动 `joint_command_interpolator` 后，链路是：

```text
上层控制节点
  -> /isaac_joint_commands
  -> joint_command_interpolator
  -> /isaac_joint_commands_interpolated
  -> Isaac Sim ROS2SubscribeJointState
```

插值节点会合并来自多个 publisher 的部分 `JointState`，并按固定频率把当前位置逐步推向最新目标。这样 `/isaac_joint_commands` 仍然是对外接口，Isaac 只接收合并后的平滑命令，避免单次目标跳变或多个部分消息互相打断。

常用关节名：

```text
底盘轮组:
fl_steering_joint, fr_steering_joint, rl_steering_joint, rr_steering_joint
fl_wheel, fr_wheel, rl_wheel, rr_wheel

升降:
lifting_joint

左臂:
left_joint1, left_joint2, left_joint3, left_joint4
left_joint5, left_joint6, left_joint7, left_joint8

右臂:
right_joint1, right_joint2, right_joint3, right_joint4
right_joint5, right_joint6, right_joint7, right_joint8
```

示例：控制左臂部分关节。

```bash
ros2 topic pub --once /isaac_joint_commands sensor_msgs/msg/JointState \
  "{name: [left_joint4, left_joint6, left_joint7, left_joint8], position: [0.3, -0.2, 0.03, -0.03]}"
```

约定：

- Revolute joint 的 `position` 单位是 rad。
- Prismatic joint 的 `position` 单位是 m。
- 如果同时填写 `velocity`，插值节点会把该速度字段转发到内部平滑 topic；当前项目中轮子视觉同步节点会给轮子 joint 填 velocity。
- 上层功能直接发布 `/isaac_joint_commands` 时，仍然要避免多个节点同时控制同一个关节；插值节点能合并不同关节的部分消息，但同一个关节的最新目标会覆盖旧目标。

### `/joint_states`

`/joint_states` 是当前最重要的状态反馈接口。

```bash
ros2 topic echo /joint_states
```

当前场景把 PhysX `timeStepsPerSecond` 设为 `200`。`ROS2PublishJointState` 仍由稳定的 `OnPlaybackTick` 触发；实际 `/joint_states` 频率需要用下面命令实测：

```bash
ros2 topic hz /joint_states
```

下游功能可以用它读取：

- 轮子转向角和轮子转动状态。
- `lifting_joint` 升降位置。
- 左右机械臂 1-8 号关节位置。
- 关节速度和 effort。

### `/tf`

如果 Isaac/场景中启用了 TF 发布，`/tf` 可用于外部功能读取坐标树。当前控制流程不依赖 `/tf`，需要时用下面命令确认：

```bash
ros2 topic info /tf
```

### 相机和场景附加话题

当前场景里还可能出现相机或数据记录相关 topic，例如：

```text
/record/camera_rgb
/record/left_camera_rgb
/record/right_camera_rgb
/record/static_info
/events/write_split
```

这些 topic 来自场景或其他扩展，不是底盘/机械臂控制的必需接口。实际类型和是否存在以运行时为准：

```bash
ros2 topic list -t
ros2 topic info /record/left_camera_rgb
```

### 推荐给后续功能的接口选择

- 移动底盘：发布 `/cmd_vel`。
- 控制单个或多个机械臂关节：发布 `/isaac_joint_commands`。
- 读取机器人关节状态：订阅 `/joint_states`。
- 需要仿真时间同步：订阅 `/clock` 并启用 `use_sim_time`。
- 需要图像输入：先用 `ros2 topic list -t` 确认可用相机 topic，再按实际类型订阅。

## 常用调用命令

### 发送底盘命令

容器内执行：

```bash
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.2, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.3}}"
```

停止：

```bash
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"
```

### 机械臂测试

启动 launch 时带参数：

```bash
./scripts/run_ros2_humble_docker.sh --launch-control -- start_arm_demo:=true arm_side:=both
```

或在容器内单独运行：

```bash
ros2 run mobile_aloha_isaac_control arm_joint_command_demo --ros-args -p side:=left
ros2 run mobile_aloha_isaac_control arm_joint_command_demo --ros-args -p side:=right
```

默认 `motion:=wave`，会围绕当前初始姿态做小幅摆动。只想发布一次零位附近安全姿态时：

```bash
ros2 run mobile_aloha_isaac_control arm_joint_command_demo --ros-args \
  -p side:=both -p motion:=safe -p publish_once:=true
```

## 控制链路

```text
ROS2 /cmd_vel
  -> Isaac Sim mobile_aloha_kinematic_base_controller.py
  -> 直接更新机器人根 Xform 的 XY/yaw

ROS2 /cmd_vel
  -> Docker cmd_vel_to_joint_command
  -> /isaac_joint_commands
  -> Docker joint_command_interpolator
  -> /isaac_joint_commands_interpolated
  -> Isaac Sim ROS2SubscribeJointState
  -> IsaacArticulationController
  -> 轮子转向/转动视觉同步

ROS2 arm_joint_command_demo
  -> /isaac_joint_commands
  -> Docker joint_command_interpolator
  -> /isaac_joint_commands_interpolated
  -> Isaac Sim ROS2SubscribeJointState
  -> IsaacArticulationController
  -> 机械臂关节测试动作

Mobile ALOHA articulation
  -> Isaac Sim ROS2PublishJointState
  -> /joint_states
```

底盘真实位移不是靠轮地摩擦驱动，而是运动学控制机器人根 Xform。这样避开了当前 URDF/USD 在全动力学轮地接触下容易倾斜、弹跳、飞天的问题。

## 关键文件

```text
assets/4.mobile_aloha_in_market.usda
dev_ws/scripts/run_isaac_mobile_aloha.sh
dev_ws/scripts/run_ros2_humble_docker.sh
dev_ws/isaac/scripts/mobile_aloha_kinematic_base_controller.py
dev_ws/isaac/scripts/create_mobile_aloha_ros_control_graph.py
dev_ws/ros2_ws/src/mobile_aloha_isaac_control/
```

ROS2 包内主要文件：

```text
config/mobile_aloha_joints.yaml
mobile_aloha_isaac_control/cmd_vel_to_joint_command.py
mobile_aloha_isaac_control/arm_joint_command_demo.py
launch/mobile_aloha_control.launch.py
```

`create_mobile_aloha_ros_control_graph.py` 不是运行时控制器，只用于重建或修复 USD 中的 Action Graph。

## 修改内容与原因

### 启动流程

新增 `dev_ws/scripts/run_isaac_mobile_aloha.sh`。

原因：之前每次都要手动设置 Isaac ROS2 环境变量，再在 Script Editor 运行运动学底盘脚本，容易漏掉。漏掉后 `/cmd_vel` 只有 Docker 侧一个 subscriber，轮子会动但车身不会真实移动。

现在该脚本会自动：

- 启动 Isaac Sim。
- 设置 ROS2 Bridge 相关环境变量。
- 通过 `--exec` 自动执行 `mobile_aloha_kinematic_base_controller.py`。

### 运动学底盘控制器

修改 `dev_ws/isaac/scripts/mobile_aloha_kinematic_base_controller.py`。

原因：Isaac 启动时 USD 可能还没打开，如果脚本立即查找机器人 prim 会失败。

现在控制器会：

- 等待 `/World/split_aloha_rslidar_with_piper_isaac_material` 出现。
- 自动订阅 `/cmd_vel`。
- Play 后根据 `linear.x`、`linear.y`、`angular.z` 更新机器人根 Xform。
- 保持 Z 高度固定，只更新 XY 和 yaw。

### ROS2 Docker 脚本

修改 `dev_ws/scripts/run_ros2_humble_docker.sh`。

原因：原来只进入容器，用户还要手动 source 环境、启动 launch。后来尝试后台启动 launch，但日志不可见，而且容易产生重复进程。最终改为前台启动。

现在脚本行为：

- 默认镜像为 `mobile_aloha_ros2_humble:latest`。
- 可用 `ISAAC_ROS2_IMAGE=...` 临时覆盖镜像。
- 容器使用 host 网络，保证 Docker 和宿主机/Isaac Sim ROS2 互通。
- `--launch-control` 会前台启动 `mobile_aloha_control.launch.py`。
- 如果已有控制 launch 运行，再次执行脚本只进入 shell，不重复启动节点。
- 如果缺少 `ros-humble-rmw-cyclonedds-cpp`，会自动安装。

### ROS2 控制包

新增/修改 `mobile_aloha_isaac_control` 包。

主要节点：

- `cmd_vel_to_joint_command`：订阅 `/cmd_vel`，发布 `/isaac_joint_commands`，用于轮子转向/转动视觉同步。
- `joint_command_interpolator`：订阅 `/isaac_joint_commands`，合并并插值后发布 `/isaac_joint_commands_interpolated` 给 Isaac。
- `arm_joint_command_demo`：发布机械臂测试 JointState，默认做小幅 `wave` 动作。

修改原因：

- Isaac 的 `IsaacArticulationController` 接收 `sensor_msgs/JointState` 更直接。
- 底盘真实移动已经由 Isaac 侧运动学脚本处理，轮子 joint 命令只负责视觉同步。
- 机械臂 demo 之前发布的 safe pose 和初始姿态太接近，看不出效果；现在默认 `wave`。

### USD 场景

修改 `assets/4.mobile_aloha_in_market.usda`。

保留：

- `/World/MobileAlohaROSControlGraph`。
- `/World/split_aloha_rslidar_with_piper_isaac_material/root_joint` 作为 USD articulation root。
- Action Graph 的 `robotPath` / `targetPrim` 指向 `base_link`。

清理/修复：

- 清理之前为轮地动力学尝试加入的高摩擦地面、额外轮子碰撞、防倾块、质量分布覆盖和强 drive 覆盖。
- 删除错误的 `box_link.xformOp:orient = (1, 0, 0, 0)` 覆盖，避免箱体和底盘/轮子视觉关系脱开。
- 删除左右机械臂初始 target pose 覆盖，让 Play 前后机械臂初始姿态一致。
- 只加强 `lifting_joint` 的保持力，避免 Play 后升降机构因重力下沉。

原因：

- 将 `base_link` 设为 articulation root 或 kinematic articulation root 都会导致 Isaac/PhysX 识别失败、飞天或忽略 articulation。
- `root_joint` 保持 articulation root，`base_link` 作为 Action Graph 刚体入口，是当前验证稳定的组合。
- 底盘不再依赖轮地动力学，之前的摩擦/防倾/质量调参会引入更多不稳定因素。

### Docker 镜像

当前默认使用：

```text
mobile_aloha_ros2_humble:latest
```

该镜像由已配置好的 `isaac_ros2_humble` 容器提交得到，包含当前需要的 ROS2 依赖。

如需重新保存当前容器：

```bash
docker commit isaac_ros2_humble mobile_aloha_ros2_humble:latest
```

## Action Graph

当前 USD 中应有：

```text
/World/MobileAlohaROSControlGraph
```

节点：

```text
OnPlaybackTick
ReadSimTime
Context
SubscribeJointState
ArticulationController
PublishJointState
PublishClock
```

关键属性：

```text
SubscribeJointState.topicName = isaac_joint_commands_interpolated
PublishJointState.topicName = joint_states
PublishJointState.execIn = OnPlaybackTick.outputs:tick
PublishJointState.targetPrim = /World/split_aloha_rslidar_with_piper_isaac_material/base_link
ArticulationController.robotPath = /World/split_aloha_rslidar_with_piper_isaac_material/base_link
ArticulationController.targetPrim = /World/split_aloha_rslidar_with_piper_isaac_material/base_link
```

如果 graph 被删，可以重建：

```bash
./isaac-sim-standalone-5.1.0-linux-x86_64/python.sh \
  dev_ws/isaac/scripts/create_mobile_aloha_ros_control_graph.py
```

## 常见问题

### 轮子动，但车身不动

检查：

```bash
ros2 topic info /cmd_vel
```

如果 `Subscription count: 1`，通常只有 Docker 的 `cmd_vel_to_joint_command` 在线，Isaac 运动学底盘控制器没运行。

解决：

```bash
./dev_ws/scripts/run_isaac_mobile_aloha.sh
```

然后重新打开 USD、点击 Play。正常应为 `Subscription count: 2`。

### `/cmd_vel` 一直等待 subscriber

说明 Docker 控制 launch 没启动。执行：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
./scripts/run_ros2_humble_docker.sh --launch-control
```

### `/joint_states` 没有 publisher

检查：

- Isaac Sim 是否点击 Play。
- `isaacsim.ros2.bridge` 是否启用。
- Action Graph 是否存在。
- `PublishJointState.targetPrim` 是否指向 `.../base_link`。

### `/isaac_joint_commands` 或内部平滑 topic 没有 subscriber

检查：

- Isaac Sim 是否点击 Play。
- `joint_command_interpolator` 是否启动。
- `SubscribeJointState.topicName` 是否是 `isaac_joint_commands_interpolated`。
- Action Graph 是否连到 `OnPlaybackTick`。

### RMW implementation not installed

如果看到：

```text
librmw_cyclonedds_cpp.so: cannot open shared object file
```

说明当前容器缺少 CycloneDDS RMW。新脚本会自动安装；也可以手动执行：

```bash
apt-get update
apt-get install -y ros-humble-rmw-cyclonedds-cpp
source /opt/ros/humble/setup.bash
source /workspace/dev_ws/ros2_ws/install/setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

### Nested articulation roots

如果看到：

```text
UsdPhysics: Nested articulation roots are not allowed.
```

说明同一机器人层级里有多个 articulation root。当前 USD 中应只保留：

```text
/World/split_aloha_rslidar_with_piper_isaac_material/root_joint
```

`base_link` 只作为 Action Graph 的 `robotPath` / `targetPrim`，不要同时作为 articulation root。

### 宿主机 `colcon build` 权限错误

不要在宿主机编译，统一在 Docker 内构建和运行 ROS2 节点。
