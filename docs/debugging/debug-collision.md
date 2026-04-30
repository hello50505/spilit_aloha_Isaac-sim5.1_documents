# 右臂抓瓶碰撞调试

本文记录一个独立 ROS2 节点，让右臂移动到 `bottle_xpc` 附近，闭合夹爪抓起瓶子，再放到另一个位置。目标不是做稳定任务规划，而是给碰撞、刚体、夹爪接触和 IK 到位情况提供一个可重复的调试动作。

## 场景

使用场景：

```text
assets/6_aloha_in_blue_grid_ROS.usda
```

场景里的瓶子：

```text
/World/bottle_xpc
```

当前 `bottle_xpc` 已经带有 PhysX 刚体和碰撞属性：

```text
PhysicsRigidBodyAPI
PhysicsCollisionAPI
physics:approximation = "convexHull"
physics:collisionEnabled = 1
physics:mass = 0.1
xformOp:translate = (-3.1374440575812965, -0.6881664434253448, 1.1869022206133355)
xformOp:scale = (0.05, 0.06, 0.31)
```

## 控制链路

抓瓶节点不直接移动 USD 物体。它只发布：

- `/right_ee_target_pose`：右臂 TCP 目标位姿，由 `right_ee_pose_ik_controller` 解 IK。
- `/isaac_joint_commands`：只发布 `right_joint7` / `right_joint8`，用于开合夹爪。

右臂 `joint1 ~ joint6` 仍由现有 IK 控制器发布，瓶子是否被抓起取决于 Isaac Sim 里的真实碰撞、质量、摩擦和夹爪接触。

## 启动 Isaac Sim

宿主机：

```bash
cd /home/xiangpc/dataset/isaac-sim
./dev_ws/scripts/run_isaac_mobile_aloha.sh
```

打开：

```text
assets/6_aloha_in_blue_grid_ROS.usda
```

然后点击 Play。确认 Stage 里能看到：

```text
/World/bottle_xpc
/World/split_aloha_rslidar_with_piper_isaac_material
```

## 启动 ROS2 控制

建议关闭 debug teleop 和 marker publisher，避免多个节点同时控制右臂目标或夹爪：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  start_base_control:=false \
  start_ee_pose_ik:=true \
  ik_side:=right \
  start_real_ee_pose:=true \
  start_debug_teleop:=false \
  start_right_ee_target_marker:=false
```

保留这些节点：

- `joint_command_interpolator`
- `right_ee_pose_ik_controller`
- `real_ee_pose_publisher`

不要同时运行：

- `mobile_aloha_debug_teleop`
- `right_ee_target_marker_publisher`
- 其它会发布 `/right_ee_target_pose` 的脚本
- 其它会控制 `right_joint7/right_joint8` 的脚本

## 运行抓瓶脚本

另开一个容器 shell：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
./scripts/run_ros2_humble_docker.sh
```

容器内运行：

```bash
ros2 run mobile_aloha_isaac_control right_bottle_pick_place_debug
```

默认动作：

```text
打开夹爪
移动到已验证过的右臂预抓取高位点
移动到瓶子前上方
移动到抓取点
关闭夹爪
抬起
移动到放置点上方
下降
打开夹爪
后撤
```

节点会订阅 `/right_ee_current_pose_in_world`。每个移动步骤都会等待当前 TCP 靠近目标点；如果超时，会打印 warning 并进入下一步，方便继续观察碰撞状态。

## 常用参数

修改放置点：

```bash
ros2 run mobile_aloha_isaac_control right_bottle_pick_place_debug --ros-args \
  -p place_center_xyz:="[-3.10, -0.75, 1.1869]"
```

降低抓取动作高度和超时时间：

```bash
ros2 run mobile_aloha_isaac_control right_bottle_pick_place_debug --ros-args \
  -p lift_height_m:=0.16 \
  -p step_timeout_s:=8.0
```

调夹爪闭合量：

```bash
ros2 run mobile_aloha_isaac_control right_bottle_pick_place_debug --ros-args \
  -p gripper_open_m:=0.035 \
  -p gripper_closed_m:=0.0
```

只看节点会发布什么，不真正发控制命令：

```bash
ros2 run mobile_aloha_isaac_control right_bottle_pick_place_debug --ros-args \
  -p dry_run:=true
```

关键默认值：

```text
bottle_center_xyz: [-3.1374440575812965, -0.6881664434253448, 1.1869022206133355]
place_center_xyz: [-3.20, -0.86, 1.1869022206133355]
pre_grasp_xyz: [-3.159010988268375, -0.6492924815329204, 1.6948670389997829]
approach_offset_xyz: [0.0, 0.14, 0.16]
grasp_offset_xyz: [0.0, 0.0, 0.04]
place_offset_xyz: [0.0, 0.0, 0.05]
lift_height_m: 0.24
pose_tolerance_m: 0.025
step_timeout_s: 12.0
continue_on_timeout: false
require_current_pose_before_motion: true
gripper_open_m: 0.03
gripper_closed_m: 0.0
```

如果右臂接近方向不合适，优先调：

```text
pre_grasp_xyz
approach_offset_xyz
grasp_offset_xyz
target_orientation_wxyz
```

如果第一步移动就 `ik_failed` 或超时，先把 `pre_grasp_xyz` 改成当前右臂附近的一个已知可达点。可以先用红色 target marker 或 debug teleop 找到可达位姿，再把对应世界坐标填回这个参数。

当前默认姿态复用场景中右臂 target marker 附近的姿态：

```text
target_orientation_wxyz: [0.5349671220016433, -0.507183226385591, -0.49263000465165485, -0.46248354756390175]
```

## 检查 topic

确认右臂 IK 目标只有抓瓶脚本在发布：

```bash
ros2 topic info /right_ee_target_pose -v
```

确认 `/isaac_joint_commands` 上没有其它右夹爪控制脚本抢控制：

```bash
ros2 topic info /isaac_joint_commands -v
```

查看当前 TCP 反馈：

```bash
ros2 topic echo --once /right_ee_current_pose_in_world
```

如果抓瓶脚本一直打印等待当前 TCP，或 IK 日志提示：

```text
Cannot transform world target: waiting for TF world -> base_link
```

先检查 Isaac Sim 是否已经点击 Play，并确认 `/tf` 里有 `world -> base_link`：

```bash
ros2 topic echo /tf
```

这个 TF 来自 Isaac 场景里的 `ROS2PublishTransformTree`。如果没有 `base_link`，`right_ee_pose_ik_controller` 无法把世界坐标目标转换到右臂求解坐标系，`real_ee_pose_publisher` 也无法发布 `/right_ee_current_pose_in_world`。

查看 IK 状态：

```bash
ros2 topic echo /right_ee_pose_ik_controller/status
```

## 碰撞可视化

Isaac Sim 菜单在不同版本可能略有差异。你当前界面的 `Window` 菜单里能看到
`Physics Stage Settings`，但不一定会直接出现单独的 `Physics Debug` 面板。
核心目标是打开 PhysX/Physics 的 collider、rigid body 和 contact 可视化。

推荐流程：

1. 打开场景并点击 Play。
2. 优先点 viewport 左上角的显示/眼睛菜单。
3. 找 `Show by type -> Physics -> Colliders -> Selected` 或 `Show by type -> Physics -> Colliders -> All`。
4. 如果没有 `Colliders`，找 `Show by type -> Physics Mesh -> All`，它通常会显示 collision mesh 轮廓。
5. 如果需要面板式调试，再试 `Window -> Simulation -> Debug`；有些版本不叫 `Physics Debug`。
6. 在 debug 面板里找 `Collision Mesh Debug Visualization`、`Solid Mesh Collision Visualization`、`Contacts`、`Contact Points`、`Contact Normals`。
7. 在 viewport 中选中 `/World/bottle_xpc`，确认它周围出现 convex hull 碰撞轮廓。
8. 选中右夹爪相关 link，确认夹爪两个指头也有碰撞体。

调试时重点观察：

- 瓶子的 convex hull 是否包住可见瓶身。
- 夹爪指尖是否有碰撞体，而不是只有视觉 mesh。
- 夹爪闭合时是否真正接触瓶子两侧。
- 抬起时 contact points 是否跟随夹爪移动。
- 放置时瓶子和桌面/蓝色网格是否有接触点。

## 常见问题

### 瓶子完全不动

检查：

```text
physics:rigidBodyEnabled = 1
physics:kinematicEnabled = 0
physics:collisionEnabled = 1
physics:mass > 0
```

还要确认 Isaac Sim 正在 Play，而不是只打开了静态场景。

### 夹爪穿过瓶子

优先打开碰撞体可视化，确认右夹爪两个指头有 collision shape。若夹爪没有碰撞体，只调 ROS2 轨迹不会产生抓取接触。

也可以减慢动作：

```bash
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  start_base_control:=false \
  start_ee_pose_ik:=true \
  ik_side:=right \
  start_real_ee_pose:=true \
  start_debug_teleop:=false \
  joint_command_interpolator_max_step_per_s:=3.0
```

### 瓶子被弹飞

常见原因：

- 抓取点太低或太偏，夹爪从瓶子内部闭合。
- `gripper_closed_m` 太小，闭合过深。
- 右臂移动速度太快，接触冲量过大。
- 瓶子质量太小或碰撞 hull 太粗。

先尝试：

```bash
ros2 run mobile_aloha_isaac_control right_bottle_pick_place_debug --ros-args \
  -p grasp_offset_xyz:="[0.0, 0.0, 0.08]" \
  -p gripper_closed_m:=0.005 \
  -p lift_height_m:=0.14
```

### IK 到不了目标

看 IK 状态：

```bash
ros2 topic echo /right_ee_pose_ik_controller/status
```

如果持续 `ik_failed`，先把目标点改得更保守：

```bash
ros2 run mobile_aloha_isaac_control right_bottle_pick_place_debug --ros-args \
  -p approach_offset_xyz:="[0.0, 0.18, 0.22]" \
  -p grasp_offset_xyz:="[0.0, 0.02, 0.08]"
```

### 放置后瓶子弹开或倒下

检查放置点附近是否有桌面/地面 collision。打开 contact points 后观察瓶子底部是否和桌面接触。若接触点不稳定，可以把 `place_offset_xyz` 的 z 增大一点，让夹爪先在更高位置松开：

```bash
ros2 run mobile_aloha_isaac_control right_bottle_pick_place_debug --ros-args \
  -p place_offset_xyz:="[0.0, 0.0, 0.08]"
```
