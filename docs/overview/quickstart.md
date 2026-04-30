# 17 — Mobile ALOHA + Isaac Sim + MoveIt 启动总结

写给"过两个月再来跑这套东西"的自己。这一篇不展开任何原理，只
说**怎么跑、有几条路、该走哪条**。出问题的细节都散在
`6_IMport_IK.md` ~ `16_import_moveit.md` 里，按章号回去翻就是。

---

## 0. 总体架构（两个进程）

```text
Host (conda)                          Docker (ROS2 Humble)
┌────────────────────────────┐        ┌──────────────────────────────────┐
│  Isaac Sim 5.1.0           │        │  mobile_aloha_ros2_humble:moveit │
│  (standalone, GPU)         │        │                                  │
│  ./scripts/run_isaac_      │ /tf    │  ./scripts/run_ros2_humble_      │
│       mobile_aloha.sh      │ /joint │       docker.sh                  │
│                            │ states │                                  │
│  - 加载 USD 场景            │ ─────► │  ros2 launch mobile_aloha_      │
│  - 跑 ROS2 Bridge action    │ ◄───── │       isaac_control             │
│    graph (发 TF, joint_     │ /isaac_│       mobile_aloha_control      │
│    states, EE marker pose)  │ joint_ │       .launch.py                │
│  - 接收 /isaac_joint_       │ commands│                                 │
│    commands_interpolated    │        │  - move_group (TRAC-IK)          │
│                            │        │  - ee_pose_ik_controller (×2)    │
│                            │        │  - joint_command_interpolator    │
│                            │        │  - real_ee_pose_publisher        │
│                            │        │  - debug_teleop (keyboard +      │
│                            │        │    SpaceMouse)                   │
│                            │        │  - (可选) RViz2                  │
└────────────────────────────┘        └──────────────────────────────────┘
```

- Isaac 跑在**宿主机**，因为 standalone 包自带 Python/Torch/CUDA，
  不能进 docker。
- ROS2 节点全部跑在 **Docker 里**，镜像
  `mobile_aloha_ros2_humble:moveit` 里同时装了 MoveIt2 + TRAC-IK
  源码编译版 + cuRobo wheel，两套 IK 后端共存。
- 通讯走 ROS2 DDS over `--net=host`，不用做端口映射。
  `RMW_IMPLEMENTATION=rmw_cyclonedds_cpp` 是默认值。

---

## 1. 默认推荐启动（不会出错的那一条）

需要两个终端。第一次跑或者改了代码加 `--build`。

### Terminal A — Isaac Sim（宿主机，conda base 即可）

```bash
cd ~/dataset/isaac-sim
./dev_ws/scripts/run_isaac_mobile_aloha.sh
```

等到 viewport 出场景、底盘和两条胳膊都加载好就行。

### Terminal B — ROS2 控制栈（开 docker，前台跑 launch）

```bash
cd ~/dataset/isaac-sim
./dev_ws/scripts/run_ros2_humble_docker.sh --build --launch-control
```

加 `--build` 让容器内做一遍 `colcon build --symlink-install`，
之后再跑就不需要 `--build` 了，除非你改了代码（symlink-install
对 Python 改动不需要 rebuild，对 launch / config 文件需要）。

跑起来你应该看到：

- `[move_group-1] You can start planning now!`
- `[ee_pose_ik_controller-5] READY left IK (moveit): ...`
- `[ee_pose_ik_controller-6] READY right IK (moveit): ...`
- `[mobile_aloha_debug_teleop-7] Debug teleop ready: keyboard->/cmd_vel, ...`
- `[real_ee_pose_publisher-4] READY real EE pose publisher: ...`

这一条命令默认就把下面这些**都关掉了**（因为容易踩坑）：

| 项 | 默认 | 备注 |
| --- | --- | --- |
| `start_arm_demo` | false | 自动挥手 demo，调试用 |
| `start_ee_pose_fk` | false | 已废弃，cuRobo FK 老链路 |
| `start_right_ee_target_marker` | false | 用 Isaac 红方块拖目标的旧链路 |
| `start_rviz` | false | 启 RViz2 + RobotModel 可视化 |
| `start_rviz_target_marker` | false | RViz 里 6-DOF 拖拽 marker |

---

## 2. Docker 启动脚本 `run_ros2_humble_docker.sh`

### 2.1 调用方式

```bash
./dev_ws/scripts/run_ros2_humble_docker.sh [OPTIONS] [-- ROS_LAUNCH_ARGS...]
```

### 2.2 脚本自己的 OPTIONS

| flag | 作用 |
| --- | --- |
| `--launch-control` | 在容器里前台跑 `mobile_aloha_control.launch.py`，Ctrl-C 退出。如果已经有一份在另一容器里跑就只起 shell。 |
| `--build` | 进容器后先 `colcon build --symlink-install --packages-select mobile_aloha_moveit_config mobile_aloha_isaac_control`，然后再 source/launch。改了 launch / config / 新加 entry point 时必加。 |
| `--no-shell` | 跟 `--launch-control` 互斥。launch 已经在跑时不再开交互 shell，直接 exit。 |
| `-h`, `--help` | 看脚本内的 usage。 |
| `--` | 分隔符，**之后的所有参数原样喂给 `ros2 launch`**（见 §3）。 |

### 2.3 环境变量（脚本顶层）

| 变量 | 默认 | 改它的场景 |
| --- | --- | --- |
| `ISAAC_ROS2_IMAGE` | `mobile_aloha_ros2_humble:moveit` | 切回老的 cuRobo 单体镜像、或者本地构了别的 tag。 |
| `ISAAC_ROS2_CONTAINER_NAME` | `isaac_ros2_humble` | 想同时跑两份（一份 moveit，一份 curobo）做对比时，给两边起不同名字。 |
| `ISAAC_ROS2_GPUS` | `all` | 设 `none` 关掉 `--gpus`（CPU debug，cuRobo 这条路会变 100% CPU 推理；MoveIt/TRAC-IK 本来就 CPU，无影响）。 |
| `ROS_DOMAIN_ID` | `0` | 同机多套 ROS2 隔离时改。 |
| `RMW_IMPLEMENTATION` | `rmw_cyclonedds_cpp` | 想退回 FastDDS 时设 `rmw_fastrtps_cpp`，但 Isaac Sim 端用的是 FastDDS 配置文件 `humble_ws/fastdds.xml`，最好跟它一致；目前 cyclonedds 在我们这套场景下也通。 |

### 2.4 docker run 用到的核心参数（脚本里写死）

```bash
docker run -it --rm \
  --name isaac_ros2_humble \
  --gpus all \
  --net=host \                       # ROS2 DDS over host network
  --ipc=host \                       # 共享 SHM，DDS 大消息走得动
  --env=DISPLAY=$DISPLAY \           # X 转发, RViz 才能弹窗
  --env=ROS_DOMAIN_ID=0 \
  --env=RMW_IMPLEMENTATION=rmw_cyclonedds_cpp \
  --env=FASTRTPS_DEFAULT_PROFILES_FILE=/humble_ws/fastdds.xml \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v "${ISAAC_SIM_ROOT}/IsaacSim-ros_workspaces/humble_ws:/humble_ws" \
  -v "${ISAAC_SIM_ROOT}:/workspace/isaac-sim:ro" \   # URDF / mesh 只读
  -v "${ISAAC_SIM_ROOT}/dev_ws:/workspace/dev_ws" \  # 源码 + build 输出可写
  -w /workspace/dev_ws/ros2_ws \
  mobile_aloha_ros2_humble:moveit \
  bash -lc "$(container_command)"
```

每一条的意义：

- `--net=host` + `--ipc=host`：DDS 发现走 multicast + 共享内存，跨
  Host↔Container 才能不丢包。**不要换成 docker bridge，**否则 Isaac
  那侧的 TF 收不到。
- `-v ${ISAAC_SIM_ROOT}:/workspace/isaac-sim:ro`：把 host 上的
  `mobile_aloha_sim/` URDF 和 mesh 挂只读到容器，
  `move_group.launch.py` 里 URDF 的 `package://` 都被替换成
  `file:///workspace/isaac-sim/...`（见 16_import_moveit.md 那一节
  "URDF / 关节名 / mesh 加载这套坑"）。
- `-v ${ISAAC_SIM_ROOT}/dev_ws:/workspace/dev_ws`：可写挂载，
  容器内 `colcon build` 的产物落到 host 上同一份目录，
  `--symlink-install` 之后 host 编辑的 Python 立即生效。
- `-v ${ISAAC_SIM_ROOT}/IsaacSim-ros_workspaces/humble_ws:/humble_ws`：
  Isaac Sim 官方提供的 ROS2 workspace（含 Isaac msgs 等），容器内
  会 `source /humble_ws/install/local_setup.bash`。
- `--env=DISPLAY` + X11 socket 挂载：RViz2 / `xeyes` 这种 GUI
  能弹到 host 屏幕。如果 RViz 起来报 X 错，先在 host 跑
  `xhost +local:docker`（脚本里已经做过了，但偶尔需要重做）。

---

## 3. `mobile_aloha_control.launch.py` 的几条主线

`-- xxx:=yyy ...` 都是这个 launch 的参数。**所有参数都有默认值**，
绝大多数直接默认就能跑。下面只列**会真正去改**的几组。

### 3.1 IK 后端（这两条互斥地走一条）

| 启动方式 | 命令 | 说明 |
| --- | --- | --- |
| **MoveIt + TRAC-IK（推荐，默认）** | `--launch-control` 即可，无需任何 `--` 参数 | 走 `move_group` + `/compute_ik` 服务。每次解 IK 都会带完整 RobotState seed，对应 `_isaac_material` URDF + SRDF 里 `virtual_world` floating joint。`ee_pose_ik_controller` 内部把 `world` 系下的目标转给 MoveIt。 |
| **cuRobo（旧链路，opt-in）** | `--launch-control -- ik_backend:=curobo start_move_group:=false` | 跑 cuRobo Python wheel，需要 GPU；URDF 走 cuRobo 自己的 urdf_parser，不需要 mesh，靠 `mobile_aloha_isaac_control/config/curobo/mobile_aloha_{left,right}.yml` 里那一份 collision_spheres 配置。`start_move_group` 必须**显式**关掉，否则会同时拉起两个 IK 求解。 |

> 切回 cuRobo 时记得检查 `dev_ws/third_party/curobo/` 在容器内能
> 被 import（脚本会自动加 PYTHONPATH，正常情况下不用动）。

### 3.2 IK 解哪一边

```bash
-- ik_side:=left      # 只起左手 IK 控制器
-- ik_side:=right     # 只起右手
-- ik_side:=both      # 默认
```

或者整条 IK 都不起：

```bash
-- start_ee_pose_ik:=false
```

——这种情况一般是手动用 `ros2 service call /compute_ik` 调试 IK，
或者在用别的下游测试 joint command interpolator。

### 3.3 遥操方式

下表里**前两种是默认开的**（`start_debug_teleop:=true`），第三、
第四种是 opt-in。

| # | 方式 | 启动 / 关闭 | 流向 |
| --- | --- | --- | --- |
| 1 | **键盘**（基座 + 升降柱） | 默认开。`-- debug_teleop_keyboard_enabled:=false` 关 | 焦点在 launch 终端时按 `w/a/s/d/q/e` 推 `/cmd_vel`，`r/f` 增减 `lifting_joint`。`+/-` 调速度，空格/x 急停，`h` 出帮助。 |
| 2 | **SpaceMouse**（左右 EE 6-DOF） | 默认开。`-- debug_teleop_spacemouse_enabled:=false` 关。需要在另一台 / 另一个终端跑 SpaceMouse → UDP 桥（见 `8_debug_teleop.md`），目标 `0.0.0.0:15000`。短按左/右键切 active arm，长按切夹爪 | SpaceMouse → UDP → debug_teleop → `/<side>_ee_target_pose` → `ee_pose_ik_controller` → `/compute_ik` → `/isaac_joint_commands` |
| 3 | **Isaac Sim 红方块**（旧链路） | `-- start_right_ee_target_marker:=true` | Isaac 场景里有个红方块 prim，拖它会通过 ROS2 bridge 发 `PoseStamped` 给我们的 marker 节点，平滑后 publish `/right_ee_target_pose`。仅右手有，左手历史上没接。**不**跟方式 4 同时开（同一个 topic 两个 publisher 会互踩）。 |
| 4 | **RViz2 6-DOF InteractiveMarker** | `-- start_rviz:=true start_rviz_target_marker:=true` | 在 RViz 里挂左右两个能拖的球 + 6 个轴控件，松手时发 `/<side>_ee_target_pose`。**注意**：当前这个 marker 在 RViz 里能看见但拖动响应有问题（截至本文档），等修。临时建议走方式 1 + 2。 |

### 3.4 RViz2 启动

```bash
-- start_rviz:=true
```

会用 `mobile_aloha_isaac_control/config/rviz/aloha_moveit.rviz` 起
RViz2，里面已经预置了：

- `RobotModel`（订阅 `/robot_description`，由 `move_group` 发）
- 自动连带起 `joint_state_name_bridge` 把 Isaac 的下划线名
  joint_states 改成 URDF 斜线名（必须，move_group 的
  `current_state_monitor` 也吃这个）
- `robot_state_publisher` 发斜线名 TF，让 `RobotModel` 能定位
  每个 link
- `Pose` display × 2 显示 `/{left,right}_ee_current_pose_in_world`
- `MotionPlanning` display 默认关，可以勾上看 ghost arm

> RViz 里**不要按 MotionPlanning 的 Plan / Execute 按钮**：
> 我们没接 controller_manager + JointTrajectoryController，
> 没法 execute；这台机器人执行链路全部走
> `ee_pose_ik_controller` → `/isaac_joint_commands`。

---

## 4. 常见踩坑（默认已经避开，列在这里供 grep）

下面每一条都是历史上踩过的坑，**默认配置都已绕开**，但是手动改
launch 参数 / config 时容易再踩，按需要回查源章节。

| 现象 | 默认避开方式 | 详细排查 |
| --- | --- | --- |
| `Semantic description is not specified for the same robot as the URDF` | SRDF `<robot name>` 已对齐 `_isaac_material` URDF | 16_import_moveit.md "URDF / 关节名 / mesh 加载这套坑" |
| RViz 里 `Could not load resource [/workspace/...]: Unable to open file ...` | `_sanitize_urdf` 用 `file://` 而不是裸路径 | 同上 |
| `Joint 'left_joint1' not found in model` 刷屏 | `joint_state_name_bridge` 自动起，move_group 订阅 `/joint_states_for_rviz` | 同上 |
| `Failed to fetch current robot state` (RViz MotionPlanning Plan) | move_group 的 `joint_states` remap 已经从 dummy topic 改到 `/joint_states_for_rviz` | 同上 |
| cuRobo 路径下 `compute_ik error_code=-31 (NO_IK_SOLUTION)` | SRDF `virtual_world` 改成 `floating` + IK request 带 `multi_dof_joint_state` | 同上 |
| SpaceMouse 推到不可达后机械臂飞 | `mobile_aloha_debug_teleop` 现已限幅 + 当前 IK 失败 N 帧打折 | 15_debug_IK.md 方案 E |
| 升降柱在桌子附近"幽灵碰撞" | 场景层 collider，跟 IK 无关 | 14_debug_ghost_collision.md |
| Isaac Sim 启动时 import torch 失败 | `run_isaac_mobile_aloha.sh` 已 `unset PYTHONPATH/PYTHONHOME/CUDA_HOME` 等 | `run_isaac_mobile_aloha.sh` 顶部注释 |
| RViz `rviz_ee_target_marker` AttributeError | `interactive_markers` Python API 是 camelCase (`applyChanges` / `setPose`) | 同 16 章 |
| RViz `rviz_ee_target_marker` 看得见但拖不动 | **当前未解决**，方式 4 暂不可用，回退到方式 1+2 | 待修复 |

---

## 5. 一些有用的诊断命令（容器里跑）

```bash
# 看 IK 服务通不通
ros2 service call /compute_ik moveit_msgs/srv/GetPositionIK \
  '{ik_request: {group_name: right_arm, ik_link_name: right/link6, \
                  pose_stamped: {header: {frame_id: world}, \
                  pose: {position: {x: 1.0, y: -0.3, z: 1.4}, \
                         orientation: {w: 1.0}}}, \
                  timeout: {sec: 1}}}'

# 看 Isaac 发的 EE 当前位姿
ros2 topic echo /right_ee_current_pose_in_world --once

# 看 marker / target 流量
ros2 topic hz /right_ee_target_pose
ros2 topic echo /right_ee_target_pose --once

# 看 joint_state_name_bridge 工作没
ros2 topic echo /joint_states_for_rviz --once | head -20
# (应该看到 left/joint1, right/joint5 这种斜线名)

# 看 TF 树是否有 world -> base_link -> lifting_link -> left/arm_base ...
ros2 run tf2_tools view_frames
# 然后宿主 evince frames.pdf
```

---

## 6. 一句话备忘

**默认就跑这条**：

```bash
# Terminal A
./dev_ws/scripts/run_isaac_mobile_aloha.sh

# Terminal B
./dev_ws/scripts/run_ros2_humble_docker.sh --build --launch-control
```

需要 RViz 看模型 + 拖目标：

```bash
./dev_ws/scripts/run_ros2_humble_docker.sh --launch-control -- \
    start_rviz:=true
```

需要切回 cuRobo：

```bash
./dev_ws/scripts/run_ros2_humble_docker.sh --launch-control -- \
    ik_backend:=curobo start_move_group:=false
```

只关心右手：

```bash
./dev_ws/scripts/run_ros2_humble_docker.sh --launch-control -- \
    ik_side:=right
```

其余都按默认来。
