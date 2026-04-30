# 键盘关节/底盘综合调试

本文记录一个独立调试节点，用键盘直接控制：

```text
底盘 / 升降 / 左右机械臂单关节 / 夹爪
左右机械臂末端位置
```

节点通过 ROS topic 控制：

- `/cmd_vel`：底盘前后、左右、旋转。
- `/isaac_joint_commands`：升降、当前选中的机械臂关节、夹爪开关。
- `/left_ee_target_pose`、`/right_ee_target_pose`：当前选中机械臂末端位置目标。

## 使用场景

用于绕开 IK 和 SpaceMouse，直接确认：

- Isaac Action Graph 是否能接收右臂关节命令。
- 左右机械臂各关节方向、范围和响应是否正常。
- 左右机械臂末端 IK 位置控制方向是否正常。
- 升降、夹爪和底盘 topic 控制链路是否正常。
- 某个关节是否存在 drive/limit/命名问题。

调试时不要同时启动会控制右臂的其它节点，例如：

- `right_ee_pose_ik_controller`
- `mobile_aloha_debug_teleop` 的右臂 SpaceMouse 控制
- `arm_joint_command_demo`

否则多个节点会同时向 `/isaac_joint_commands` 写同一批关节，现象会互相抢控制。底盘调试时也不要同时运行其它 `/cmd_vel` 发布器。

如果要调试末端位置控制，需要保持对应 IK 节点运行。也就是不要关闭 `start_ee_pose_ik`，或者至少确认 `/left_ee_target_pose` / `/right_ee_target_pose` 有 IK subscriber。

## 启动 Isaac Sim

宿主机：

```bash
cd /home/xiangpc/dataset/isaac-sim
./dev_ws/scripts/run_isaac_mobile_aloha.sh
```

打开 `assets/5.mobile_aloha_in_market_ros_camera.usda` 并点击 Play。

## 启动基础 ROS2 控制

如果只想做关节、升降、夹爪和底盘调试，建议关闭 IK 和 debug teleop：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  start_base_control:=false \
  start_ee_pose_ik:=false \
  start_debug_teleop:=false
```

这样会保留默认的 `joint_command_interpolator`，但关闭其它会向 `/isaac_joint_commands` 发布目标的控制节点，隔离测试这个接口是否能稳定控制 `right_joint2`。

如果要同时调试末端位置键盘控制，请保留 IK：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  start_base_control:=false \
  start_debug_teleop:=false \
  start_ee_pose_ik:=true \
  ik_side:=both
```

不要使用默认 `--launch-control` 来做单关节接口测试，因为默认会启动：

- `cmd_vel_to_joint_command`：50Hz 向 `/isaac_joint_commands` 发布轮子关节目标。
- 左右 IK：向 `/isaac_joint_commands` 发布手臂关节目标。
- debug teleop：向 `/isaac_joint_commands` 发布升降/夹爪目标。

当前默认链路中，Isaac 不再直接订阅 `/isaac_joint_commands`，而是订阅插值后的 `/isaac_joint_commands_interpolated`。`joint_command_interpolator` 会合并多个部分 `JointState`，并把跳变目标变成连续小步进。

## 启动键盘调试节点

另开一个容器 shell：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
./scripts/run_ros2_humble_docker.sh
```

容器内运行：

```bash
ros2 run mobile_aloha_isaac_control right_joint2_keyboard_control
```

节点启动时会先打印完整按键说明。节点也会检查 `/isaac_joint_commands` 的 publisher 数量。如果发现不止一个 publisher，会周期性打印黄字警告。现在不同关节的部分消息会由插值节点合并；但如果多个节点同时控制同一个关节，仍然会互相覆盖目标。

按键：

| 按键 | 功能 |
| --- | --- |
| `l` / `r` | 切换左臂 / 右臂 |
| `1` ~ `8` | 选择当前手臂的 `joint1` ~ `joint8` |
| `i` / 上方向键 | 当前选中关节增加 `step_rad` |
| `k` / 下方向键 | 当前选中关节减少 `step_rad` |
| `I` / `K` | 当前手臂末端沿 world `X+` / `X-` 小步移动 |
| `J` / `L` | 当前手臂末端沿 world `Y+` / `Y-` 小步移动 |
| `U` / `O` | 当前手臂末端沿 world `Z+` / `Z-` 小步移动 |
| `0` | 当前选中关节目标设为 `0.0` |
| `o` / `p` | 当前手臂夹爪打开 / 关闭 |
| `t` / `g` | 升降上升 / 下降 |
| `w` / `s` | 底盘前进 / 后退 |
| `a` / `d` | 底盘左移 / 右移 |
| `q` / `e` | 底盘左转 / 右转 |
| `space` / `x` | 底盘停止 |
| `+` / `-` | 调整底盘速度倍率 |
| `h` | 打印帮助 |
| `Ctrl+C` | 退出 |

默认参数：

```text
joint_name: right_joint2
step_rad: 0.05
ee_step_m: 0.02
lift_step_m: 0.02
lift_min_m: -0.60
lift_max_m: 0.0
gripper_open_m: 0.02
gripper_closed_m: 0.0
base_speed_scale: 0.5
keyboard_poll_rate_hz: 30.0
```

机械臂关节限位来自当前 URDF：

```text
joint1: [-2.618, 2.618]
joint2: [0.0, 3.14]
joint3: [-2.697, 0.0]
joint4: [-1.832, 1.832]
joint5: [-1.22, 1.22]
joint6: [-3.14, 3.14]
joint7: [0.0, 0.05]
joint8: [-0.05, 0.0]
```

节点每次按 `i/k` 时都会读取最新 `/joint_states` 中当前选中关节的实际角度，计算 `当前角度 + step_rad`，然后只向 `/isaac_joint_commands` 发布一次新的绝对目标。连续平滑执行由 `joint_command_interpolator` 在 `/isaac_joint_commands_interpolated` 上完成。

节点每次按 `I/K/J/L/U/O` 时，会读取当前选中手臂的最新 `/left_ee_current_pose_in_world` 或 `/right_ee_current_pose_in_world`，在当前位置上叠加 `ee_step_m`，并发布到对应的 EE target topic。姿态保持当前末端姿态不变，只调位置。

底盘按键会发布 `/cmd_vel`。如果超过 `keyboard_timeout_s` 没有继续按底盘键，节点会自动发布零速度。

当前场景中 PhysX step 设为 200Hz，`/joint_states` 实际输出频率可以用 `ros2 topic hz /joint_states` 确认。

## 底层 Drive 参数

Isaac Sim 里 `/isaac_joint_commands` 最终会进入 `IsaacArticulationController`，底层关节响应由 USD 里的 joint drive 决定，不是 ROS2 节点里的 PID。

当前场景 `assets/5.mobile_aloha_in_market_ros_camera.usda` 已对左右臂 `joint1 ~ joint6` 显式设置 angular drive：

```text
drive:angular:physics:stiffness = 20000
drive:angular:physics:damping = 900
drive:angular:physics:maxForce = 5000
```

调参经验：

- 抖动、超调、来回震荡：先增大 `damping`，必要时降低 `stiffness`。
- 跟随太慢、目标到不了：先增大 `maxForce`，再适当增大 `stiffness`。
- 单次按键动作太冲：减小 `step_rad`，或者启动控制 launch 时降低 `joint_command_interpolator_max_step_per_s`。

单关节调试如果还觉得太冲，可以临时用较慢插值；如果觉得响应太慢，把这个值提高到 `3.0 ~ 5.0`，或者直接不覆盖使用默认值。

```bash
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  start_base_control:=false \
  start_ee_pose_ik:=false \
  start_debug_teleop:=false \
  joint_command_interpolator_max_step_per_s:=3.0
```

## 参数覆盖

减小步长：

```bash
ros2 run mobile_aloha_isaac_control right_joint2_keyboard_control --ros-args \
  -p step_rad:=0.02
```

临时修改初始选中的关节：

```bash
ros2 run mobile_aloha_isaac_control right_joint2_keyboard_control --ros-args \
  -p joint_name:=right_joint3 \
  -p step_rad:=0.03
```

## 检查

查看命令是否发布：

```bash
ros2 topic echo /isaac_joint_commands
```

检查 publisher/subscriber 数量：

```bash
ros2 topic info /isaac_joint_commands -v
ros2 topic info /isaac_joint_commands_interpolated -v
```

隔离测试时理想状态：

```text
Publisher count: 1
Subscription count: 1
```

`/isaac_joint_commands` 的唯一 publisher 应该是 `right_joint2_keyboard_control`，唯一 subscriber 应该是 `joint_command_interpolator`。`/isaac_joint_commands_interpolated` 的 publisher 应该是 `joint_command_interpolator`，subscriber 应该是 Isaac Sim 场景里的 `ROS2SubscribeJointState`。

查看关节反馈：

```bash
ros2 topic echo /joint_states
```

正常情况下，按 `i/k/t/g/o/p` 后 `/isaac_joint_commands` 会出现对应关节目标消息，`/isaac_joint_commands_interpolated` 会出现平滑后的执行消息。按 `w/a/s/d/q/e` 后 `/cmd_vel` 会出现底盘速度消息。

按 `I/K/J/L/U/O` 后，检查：

```bash
ros2 topic echo /right_ee_target_pose
ros2 topic echo /left_ee_target_pose
```

当前选中手臂对应的 topic 应出现 `frame_id: world` 的目标位姿消息。
