# 红色方块调试右臂 IK

本文记录一个可选调试功能：在 Isaac Sim 里拖动红色方块
`/World/right_ee_target_marker`，由 ROS2 节点把方块世界位姿发布到
`/right_ee_target_pose`，驱动右臂 IK。

默认启动控制链路时不启用这个调试节点，避免普通控制流程被红色方块影响。

## 启动

先启动 Isaac Sim 并打开场景：

```bash
cd /home/xiangpc/dataset/isaac-sim
./dev_ws/scripts/run_isaac_mobile_aloha.sh
```

在 Isaac Sim 中打开 `assets/5.mobile_aloha_in_market_ros_camera.usda`，确认
`isaacsim.ros2.bridge` 已启用并点击 Play。

普通控制启动方式不启动红方块调试：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
ISAAC_ROS2_IMAGE=mobile_aloha_ros2_humble:curobo \
  ./scripts/run_ros2_humble_docker.sh --build --launch-control
```

需要启用红方块调试时显式打开：

```bash
ISAAC_ROS2_IMAGE=mobile_aloha_ros2_humble:curobo \
  ./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  start_right_ee_target_marker:=true
```

## 当前约定

- 红色方块是无碰撞可视化目标，路径为 `/World/right_ee_target_marker`。
- Isaac 启动脚本第一次收到 `/right_ee_current_pose_in_world` 后，会把红色方块动态对齐到当前右臂 TCP 位姿；之后不再自动跟随，方便手动拖拽。
- 方块 TF 由 Isaac Sim 的 `ROS2PublishTransformTree` 发布。
- `right_ee_target_marker_publisher` 会等红色方块和当前 TCP 对齐后才允许发布 `/right_ee_target_pose`，避免启动时因为红方块旧位姿给 IK 发错误目标。
- 对齐完成后，`right_ee_target_marker_publisher` 读取 `world -> right_ee_target_marker`，发布 `/right_ee_target_pose`。
- `/right_ee_target_pose.header.frame_id` 为 `world`。
- 目标位置来自方块世界坐标，目标姿态来自方块世界旋转。
- 发布器默认做位置平滑：`8 Hz`、`0.008 m` 位置死区、`0.025 m` 单步上限。
- 场景 USD 给 `joint7` / `joint8` 加了 prismatic drive，保持夹爪开口，避免动态 TCP 因夹爪自由滑动而抖。

## 调试命令

查看红方块目标是否输出：

```bash
ros2 topic echo /right_ee_target_pose
```

查看 IK 状态：

```bash
ros2 topic echo /right_ee_pose_ik_controller/status
```

查看当前右臂 TCP 世界位姿：

```bash
ros2 topic echo --once /right_ee_current_pose_in_world
```

查看必要 TF：

```bash
ros2 topic echo /tf
```

应至少能看到：

```text
world -> base_link
world -> right_ee_target_marker
right_link6 -> right_link7
right_link6 -> right_link8
```

## 常见问题

### `/right_ee_target_pose` 没有输出

检查是否显式启动了调试节点：

```bash
start_right_ee_target_marker:=true
```

还要确认场景已 Play，且 `/tf` 中有 `world -> right_ee_target_marker`，同时 `/right_ee_current_pose_in_world` 已经有输出。发布器默认会等待红色方块先对齐当前 TCP。

### IK 偶发失败

常见原因是目标位姿不可达、处于奇异位形附近，或姿态约束过强。当前可以通过
`mobile_aloha_joints.yaml` 调整：

```yaml
ee_pose_control:
  position_tolerance_m: 0.005
  rotation_tolerance_rad: 0.10
```

现在默认会约束末端姿态，`0.10 rad` 约等于 5.7 度。如果只关心位置，可以临时调大 `rotation_tolerance_rad`；如果要控制方块或 SpaceMouse 旋转带来的末端姿态，先从当前 TCP 附近小幅旋转。

### 运动仍然抖

优先小步拖动方块。需要更慢时可以降低 marker 发布速度：

```bash
ISAAC_ROS2_IMAGE=mobile_aloha_ros2_humble:curobo \
  ./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  start_right_ee_target_marker:=true \
  target_marker_publish_rate_hz:=5.0 \
  target_marker_max_step_m:=0.015 \
  target_marker_position_epsilon_m:=0.012
```
