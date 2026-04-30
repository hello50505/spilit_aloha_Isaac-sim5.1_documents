# 键盘 + SpaceMouse 调试控制

本文记录 Mobile ALOHA 的人工调试遥操作流程：

- 键盘控制底盘 `/cmd_vel`。
- 键盘控制升降 `lifting_joint`。
- SpaceMouse 控制左/右机械臂 TCP 目标位姿。
- SpaceMouse 按钮短按切换机械臂，长按控制夹爪开/关。

## 启动 Isaac Sim

宿主机终端：

```bash
cd /home/xiangpc/dataset/isaac-sim
./dev_ws/scripts/run_isaac_mobile_aloha.sh
```

在 Isaac Sim 中打开 `assets/6_aloha_in_blue_grid_ROS.usda`，确认
`isaacsim.ros2.bridge` 已启用并点击 Play。

## 启动 ROS2 控制和调试节点

另开宿主机终端：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
./scripts/run_ros2_humble_docker.sh --build --launch-control
```

默认会使用 `mobile_aloha_ros2_humble:curobo` 镜像，并启动：

- `start_ee_pose_ik:=true`
- `ik_side:=both`
- `start_debug_teleop:=true`
- `debug_teleop_arm_mode:=both`

默认进程和绿色 READY 编号：

| 编号 | 节点 | 就绪条件 |
| --- | --- | --- |
| `[0]` | `joint_command_interpolator` | 已把 `/isaac_joint_commands` 接到 `/isaac_joint_commands_interpolated`，且 Isaac 正在订阅内部平滑 topic |
| `[1]` | `cmd_vel_to_joint_command` | 已订阅 `/cmd_vel`，且 `/isaac_joint_commands` 有下游 subscriber |
| `[2]` | `real_ee_pose_publisher` | `/tf` 中已有左右 `link7/link8`，并开始发布当前 TCP 世界位姿 |
| `[3]` | `left_ee_pose_ik_controller` | 左臂 `/joint_states`、`/tf`、手指 TF、`/isaac_joint_commands` 都就绪 |
| `[4]` | `right_ee_pose_ik_controller` | 右臂 `/joint_states`、`/tf`、手指 TF、`/isaac_joint_commands` 都就绪 |
| `[5]` | `mobile_aloha_debug_teleop` | 键盘/SpaceMouse 输入初始化，目标 topic 和当前末端位姿 topic 已连接 |

看到 `[0]` 到 `[5]` 的绿色 `READY` 后，键盘、升降、SpaceMouse 和 IK 链路才算都进入可用状态。

键盘焦点放在这个 ROS2 控制终端里。

如果只想测试键盘底盘和升降，不接 SpaceMouse：

```bash
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  debug_teleop_spacemouse_enabled:=false
```

## 启动宿主机 SpaceMouse Bridge

宿主机另开终端：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
python3 scripts/run_spacemouse_bridge.py
```

如果缺依赖：

```bash
python3 -m pip install pyspacemouse hidapi
```

默认 bridge 会把 SpaceMouse 数据发到 `udp://127.0.0.1:15000`。ROS2 Docker 使用
`--net=host`，所以 Docker 内调试节点可以直接监听这个 UDP 端口。

如果需要确认 SpaceMouse 的实际轴方向，先不要启动控制，运行校准模式：

```bash
python3 scripts/run_spacemouse_bridge.py --calibrate
```

然后一次只做一个动作，例如前推、右推、上推、roll、pitch、yaw。终端会显示主导轴：

```text
trans=y- raw_t=(+0.00,-0.82,+0.01)
rot=pitch+ raw_r=(+0.02,+0.76,-0.01)
```

把每个动作对应的主导轴和正负号记录下来，再调整 `debug_teleop_space_world_*_axis/sign` 或旋转映射。

## 键盘控制

| 按键 | 功能 |
| --- | --- |
| `w` / `s` | 底盘前进 / 后退 |
| `a` / `d` | 底盘左移 / 右移 |
| `q` / `e` | 底盘左转 / 右转 |
| `r` / `f` | 升降上升 / 下降 |
| `+` / `-` | 调整底盘速度倍率 |
| `space` 或 `x` | 底盘急停 |
| `h` | 打印帮助 |
| `Ctrl+C` | 退出节点，退出前发布零 `/cmd_vel` |

底盘命令会持续发布到 `/cmd_vel`。如果键盘一段时间没有输入，节点会自动把底盘速度归零。

升降命令直接发布到 `/isaac_joint_commands`，消息里只包含 `lifting_joint`。默认会先经过 `joint_command_interpolator`，再由 `/isaac_joint_commands_interpolated` 平滑送进 Isaac。

默认升降范围来自机器人引用 URDF 里的 `lifting_joint` limit：`-0.60 ~ 0.0 m`，每次按键步长是 `0.02 m`。需要临时调整时：

```bash
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  debug_teleop_lift_min_m:=-0.60 \
  debug_teleop_lift_max_m:=0.0 \
  debug_teleop_lift_step_m:=0.03
```

## SpaceMouse 控制

SpaceMouse 平移轴映射为当前 active arm 的 TCP 在 `world` 坐标系下的小步位置增量。

### 位置控制实现原理

宿主机 bridge 读取 SpaceMouse 原始状态，并通过 UDP 发给 ROS2 调试节点：

```text
translation: [x, y, z]
rotation: [roll, pitch, yaw]
buttons: [left, right]
```

ROS2 调试节点 `mobile_aloha_debug_teleop` 做三件事：

1. 先读取当前末端真实世界位姿。该位姿由 `real_ee_pose_publisher` 根据 `/tf` 中 `link7/link8` 的世界位姿计算，不再使用 cuRobo FK。
2. 把 SpaceMouse 当前输入换算成本周期 `world` 坐标系下的小位移。
3. 发布 `current_world_pose + delta_this_tick` 作为新的 IK 目标。

关键点是：目标不是长期积分出来的路径点，而是每次都基于最新当前末端位姿加本周期增量：

```text
target_position_world = latest_current_tcp_position_world + delta_world_this_tick
target_orientation_world = latest_current_tcp_orientation_world (+ optional rotation_delta)
```

这样 IK 如果一帧没跟上，下一次目标仍然贴着当前末端附近，不会越积越远。

调试节点会先订阅：

```text
/left_ee_current_pose_in_world
/right_ee_current_pose_in_world
```

拿到当前末端世界坐标后，把 SpaceMouse 推动方向转换成 `world` 下的 `x/y/z` 小步增量。每次发布都用最新当前末端位姿作为基准，只加本周期增量，避免 IK 跟不上时目标越积越远。然后发布：

```text
/left_ee_target_pose
/right_ee_target_pose
```

发布的 `PoseStamped.header.frame_id` 默认为 `world`，由现有 IK 节点转换到对应手臂的 `arm_base` 后求解。

当前平移映射的计算等价于：

```text
delta_world_x = spacemouse_y * space_world_x_sign * space_linear_scale_mps * dt
delta_world_y = spacemouse_x * space_world_y_sign * space_linear_scale_mps * dt
delta_world_z = spacemouse_z * space_world_z_sign * space_linear_scale_mps * dt
```

默认参数是：

```text
space_world_x_axis = axis_y
space_world_y_axis = axis_x
space_world_z_axis = axis_z
space_world_x_sign = -1.0
space_world_y_sign = 1.0
space_world_z_sign = 1.0
space_linear_scale_mps = 1.20
```

所以当前默认行为是：

```text
前推 Y+ -> world X-
右推 X+ -> world Y+
上拉 Z+ -> world Z+
```

场景 `assets/6_aloha_in_blue_grid_ROS.usda` 里还有两个无碰撞小方块：

```text
/World/right_ee_target_pose_marker
/World/right_ee_current_pose_marker
```

Isaac 启动脚本会订阅对应 topic 并实时更新它们：

- 红色方块 `/World/right_ee_target_marker`：可拖拽 marker 调试目标。
- 绿色方块 `/World/right_ee_target_pose_marker`：ROS2 实际发布到 `/right_ee_target_pose` 的 IK 目标。
- 黄色方块 `/World/right_ee_current_pose_marker`：`/right_ee_current_pose_in_world` 当前真实 TCP 位姿。

默认 SpaceMouse 平移灵敏度是 `1.20 m/s`，旋转灵敏度是 `1.50 rad/s`。

为了减少机械臂遥操作卡顿，当前默认频率为：

- SpaceMouse bridge：`60 Hz`
- debug teleop 主循环：`60 Hz`
- SpaceMouse 目标 Pose 发布：`12 Hz`
- FK 当前末端位姿发布：`50 Hz`
- IK 关节命令发布：`50 Hz`

IK 每周期最大关节步进为 `0.06 rad`。IK 目标订阅队列只保留最新一条，避免 SpaceMouse 目标积压后继续处理过期位姿。

如果 IK 日志显示单次求解在 `55-80 ms`，有效吞吐只有约 `12-18 Hz`，所以目标 Pose 默认限制到 `12 Hz`。需要调试时可以覆盖：

```bash
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  debug_teleop_target_publish_rate_hz:=8.0
```

默认方向映射：

- SpaceMouse 前推 `Y+`：world `X-`。
- SpaceMouse 右推 `X+`：world `Y+`。
- SpaceMouse 上拉 `Z+`：world `Z+`。
- SpaceMouse 右压 `roll+`：目标 roll 正方向。
- SpaceMouse 前压 `pitch+`：目标 pitch 负方向。
- SpaceMouse 绕竖直轴右转 `yaw+`：目标 yaw 负方向。

需要继续调大时：

```bash
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  debug_teleop_space_linear_scale_mps:=1.80 \
  debug_teleop_space_angular_scale_radps:=2.00
```

如果不同设备方向相反，可以只改对应符号。例如把前后方向反过来：

```bash
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  debug_teleop_space_world_x_sign:=1.0
```

如果设备轴定义和默认不同，也可以改轴映射。例如切回 SpaceMouse `x` 轴映射到 world `X`：

```bash
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  debug_teleop_space_world_x_axis:=axis_x \
  debug_teleop_space_world_y_axis:=axis_y
```

IK 节点收到目标后只打印一行结果：绿色表示解算成功，红色表示失败，并显示本次 IK 解算耗时。

默认启用 SpaceMouse 姿态控制，旋转轴会作为本周期姿态增量叠加到最新当前末端姿态上。IK 姿态容忍度为 `0.10 rad`。如果只想临时测试位置控制，可以关闭姿态输入：

```bash
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  debug_teleop_enable_rotation:=false
```

按钮规则：

| 操作 | 功能 |
| --- | --- |
| 左按钮短按 | 切到左臂 |
| 右按钮短按 | 切到右臂 |
| 左右按钮同时短按 | 切到双臂 |
| 左按钮长按 | 左夹爪开/关切换 |
| 右按钮长按 | 右夹爪开/关切换 |

夹爪不走 IK。调试节点直接发布：

```text
left_joint7, left_joint8
right_joint7, right_joint8
```

## 检查 Topic

容器内检查：

```bash
ros2 topic info /cmd_vel
ros2 topic info /isaac_joint_commands
ros2 topic info /isaac_joint_commands_interpolated
ros2 topic info /left_ee_target_pose
ros2 topic info /right_ee_target_pose
ros2 topic echo /left_ee_pose_ik_controller/status
ros2 topic echo /right_ee_pose_ik_controller/status
```

正常情况：

- `/cmd_vel` 有调试节点 publisher，并有 Isaac 底盘控制器和 `cmd_vel_to_joint_command` 两个 subscriber。
- `/left_ee_target_pose`、`/right_ee_target_pose` 有调试节点 publisher，左右 IK 节点是 subscriber。
- `/isaac_joint_commands` 里能看到轮子、升降、夹爪和 IK 节点分别发布的目标关节命令。
- `/isaac_joint_commands_interpolated` 里能看到插值节点合并后的平滑关节命令，Isaac 订阅这个内部 topic。

## 常见问题

### 键盘没反应

确认键盘焦点在启动 `start_debug_teleop:=true` 的 ROS2 控制终端里。如果通过非交互终端启动，节点可能无法打开 `/dev/tty`，这时日志会提示 `Keyboard disabled`。

### SpaceMouse 没反应

先在宿主机运行：

```bash
python3 scripts/run_spacemouse_bridge.py --print-packets
```

如果没有数据，检查 SpaceMouse USB 权限或 `pyspacemouse/hidapi` 安装。

### IK 偶发失败

优先降低 SpaceMouse 移动幅度，小步靠近目标。当前调试节点默认保持姿态不变，先验证位置控制；如果打开了旋转轴，目标姿态更容易不可达。
