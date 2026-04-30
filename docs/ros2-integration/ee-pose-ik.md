# Isaac Sim ROS2 末端位姿 IK 控制

本文记录在场景 5 上测试 Mobile ALOHA 机械臂末端位姿 IK 控制的流程。

当前实现：

- ROS2 可执行节点：`ee_pose_ik_controller`
- 默认启动两个节点实例：`left_ee_pose_ik_controller` 和 `right_ee_pose_ik_controller`
- 当前末端位姿状态节点：`ee_pose_fk_publisher`
- IK 后端：cuRobo Python API
- 输入：`geometry_msgs/msg/PoseStamped`，话题为 `/left_ee_target_pose` 和 `/right_ee_target_pose`
- IK 输入支持 `frame_id` 为对应 `arm_base`、`base_link` 或 `world`；节点内部会统一转换到对应手臂的 `arm_base` 后求解。
- 输出：`sensor_msgs/msg/JointState` 到 `/isaac_joint_commands`
- 状态反馈：订阅 Isaac 发布的 `/joint_states`
- 机械臂底座坐标系末端位姿输出：`/left_ee_current_pose_in_arm_base` 和 `/right_ee_current_pose_in_arm_base`
- 统一底盘坐标系末端位姿输出：`/left_ee_current_pose_in_base` 和 `/right_ee_current_pose_in_base`
- 世界坐标系末端位姿输出：`/left_ee_current_pose_in_world` 和 `/right_ee_current_pose_in_world`

对外的 `ee` 位姿语义是动态 TCP：TCP 位置是夹爪 `link7` 和 `link8` 当前位姿的中点，TCP 姿态跟随 `link8`。cuRobo 内部仍以 `left/link6` / `right/link6` 为 tool frame 求解，节点会自动在 TCP 和 `link6` 之间转换。

第一版只做 IK，不做世界碰撞规划。目标位姿应先从靠近当前 TCP 姿态的小位移开始测试。

## 关键文件

```text
assets/5.mobile_aloha_in_market_ros_camera.usda
dev_ws/scripts/run_isaac_mobile_aloha.sh
dev_ws/scripts/run_ros2_humble_docker.sh
dev_ws/ros2_ws/src/mobile_aloha_isaac_control/config/mobile_aloha_joints.yaml
dev_ws/ros2_ws/src/mobile_aloha_isaac_control/config/curobo/mobile_aloha_left.yml
dev_ws/ros2_ws/src/mobile_aloha_isaac_control/config/curobo/mobile_aloha_right.yml
dev_ws/ros2_ws/src/mobile_aloha_isaac_control/mobile_aloha_isaac_control/ee_pose_ik_controller.py
dev_ws/ros2_ws/src/mobile_aloha_isaac_control/launch/mobile_aloha_control.launch.py
```

## 控制链路

```text
/left_ee_target_pose or /right_ee_target_pose
  -> left_ee_pose_ik_controller or right_ee_pose_ik_controller
  -> transform target pose to left/right arm_base when frame_id is base_link or world
  -> convert target TCP pose to target link6 pose
  -> cuRobo IKSolver
  -> /isaac_joint_commands
  -> Isaac Sim ROS2SubscribeJointState
  -> IsaacArticulationController
  -> Mobile ALOHA arm joints

Isaac Sim ROS2PublishJointState
  -> /joint_states
  -> left_ee_pose_ik_controller / right_ee_pose_ik_controller seed state
  -> ee_pose_fk_publisher
  -> /left_ee_current_pose_in_arm_base and /right_ee_current_pose_in_arm_base
  -> /left_ee_current_pose_in_base and /right_ee_current_pose_in_base

Isaac Sim ROS2PublishTransformTree
  -> /tf: world -> base_link and link6 -> link7/link8
  -> ee_pose_fk_publisher
  -> /left_ee_current_pose_in_world and /right_ee_current_pose_in_world
```

## cuRobo 安装约定

cuRobo 源码放在宿主机本地：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
mkdir -p third_party
cd third_party
git clone https://github.com/NVlabs/curobo.git
```

进入 ROS2 Docker 后编译安装。这样生成的 CUDA/PyTorch 扩展会匹配 ROS2 节点实际运行的 Python 环境：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
./scripts/run_ros2_humble_docker.sh
```

容器内执行：

```bash
apt update
apt install -y git-lfs python3-pip
git lfs install
python3 -m pip install "setuptools_scm<8" wheel ninja tomli
python3 -m pip install torch --index-url https://download.pytorch.org/whl/cu124
python3 -m pip install \
  importlib_resources matplotlib networkx numpy-quaternion pyyaml scipy tqdm trimesh viser yourdfpy warp-lang \
  -i https://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com
python3 -m pip install --force-reinstall "numpy<1.25" "scipy>=1.10,<1.12" \
  -i https://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com
cd /workspace/dev_ws/third_party/curobo
git lfs pull
python3 -m pip install -e . --no-build-isolation
SETUPTOOLS_SCM_PRETEND_VERSION=0.0.0 python3 -c "import curobo; print('curobo ok')"
```

如果 `import curobo` 成功，建议保存当前容器：

```bash
docker commit isaac_ros2_humble mobile_aloha_ros2_humble:curobo
```

当前已保存镜像：

```text
mobile_aloha_ros2_humble:curobo
```

说明：cuRobo 源码挂载运行时会读取 git 版本。当前 Docker 里设置
`SETUPTOOLS_SCM_PRETEND_VERSION=0.0.0`，用于绕过 `setuptools_scm` /
`vcs-versioning` 的运行时版本探测兼容问题。
当前 cuRobo 源码结构是 `third_party/curobo/curobo`，启动脚本会把
`/workspace/dev_ws/third_party/curobo` 加入 `PYTHONPATH`。

## 启动场景 5

宿主机终端执行：

```bash
cd /home/xiangpc/dataset/isaac-sim
./dev_ws/scripts/run_isaac_mobile_aloha.sh
```

Isaac Sim 打开后：

1. 打开 `assets/5.mobile_aloha_in_market_ros_camera.usda`。
2. 确认 `isaacsim.ros2.bridge` 已启用。
3. 点击 Play。

## 启动 ROS2 控制和 IK 节点

另开宿主机终端：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
ISAAC_ROS2_IMAGE=mobile_aloha_ros2_humble:curobo \
  ./scripts/run_ros2_humble_docker.sh --build --launch-control
```

`--launch-control` 默认会同时启动：

- `/cmd_vel` 到 `/isaac_joint_commands` 的底盘控制节点。
- `/joint_states` 到当前末端位姿的 FK 状态发布节点。
- 左臂 `/left_ee_target_pose` IK 节点。
- 右臂 `/right_ee_target_pose` IK 节点。

只启动单侧 IK 时可以追加 launch 参数：

```bash
ISAAC_ROS2_IMAGE=mobile_aloha_ros2_humble:curobo \
  ./scripts/run_ros2_humble_docker.sh --build --launch-control -- ik_side:=left
```

如果只想启动底盘 `/cmd_vel` 控制，不启动 IK：

```bash
ISAAC_ROS2_IMAGE=mobile_aloha_ros2_humble:curobo \
  ./scripts/run_ros2_humble_docker.sh --build --launch-control -- start_ee_pose_ik:=false
```

不要同时启动 `start_arm_demo:=true` 控制同一条机械臂。

## 检查 topic

再开一个容器 shell：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
ISAAC_ROS2_IMAGE=mobile_aloha_ros2_humble:curobo \
  ./scripts/run_ros2_humble_docker.sh
```

容器内检查：

```bash
ros2 topic info /joint_states
ros2 topic info /isaac_joint_commands
ros2 topic info /left_ee_target_pose
ros2 topic info /right_ee_target_pose
ros2 topic echo --once /left_ee_current_pose_in_arm_base
ros2 topic echo --once /right_ee_current_pose_in_arm_base
ros2 topic echo --once /left_ee_current_pose_in_base
ros2 topic echo --once /right_ee_current_pose_in_base
ros2 topic echo --once /left_ee_current_pose_in_world
ros2 topic echo --once /right_ee_current_pose_in_world
ros2 topic echo /left_ee_pose_ik_controller/status
ros2 topic echo /right_ee_pose_ik_controller/status
```

正常应看到：

- `/joint_states` 有 Isaac publisher。
- `/isaac_joint_commands` 有 `joint_command_interpolator` subscriber。
- `/isaac_joint_commands_interpolated` 有 Isaac subscriber。
- `/left_ee_target_pose` 和 `/right_ee_target_pose` 有 IK 节点 subscriber。
- `/left_ee_current_pose_in_arm_base` 和 `/right_ee_current_pose_in_arm_base` 能输出各自 `arm_base` 下的当前末端位姿。
- `/left_ee_current_pose_in_base` 和 `/right_ee_current_pose_in_base` 能输出同一个 `base_link` 下的当前末端位姿，用于比较左右臂空间位置。
- `/left_ee_current_pose_in_world` 和 `/right_ee_current_pose_in_world` 能输出 `world` 下的当前末端位姿；车体移动时这两个 topic 会跟随变化。

## 发送左臂目标位姿

先发一个靠近当前左臂 TCP 的目标。目标姿态建议直接复制当前 TCP 姿态，不要先用 `{w: 1.0, x: 0.0, y: 0.0, z: 0.0}` 这种 identity 姿态测试，6 自由度机械臂可能会因为位姿组合过约束而无解。

IK 目标可以用三种坐标系：

- `frame_id: left/arm_base` 或 `right/arm_base`：对应手臂局部坐标系下的 TCP 目标，最适合底层调试。
- `frame_id: base_link`：节点会用 URDF 固定变换和当前 `lifting_joint` 转到对应 `arm_base`。
- `frame_id: world`：节点会先读取 `/tf` 的 `world -> base_link`，再转到对应 `arm_base`。

建议先看当前 FK 位姿，再在当前 pose 附近小步调整目标。局部 IK 调试看：

```bash
ros2 topic echo --once /left_ee_current_pose_in_arm_base
```

如果要发 `base_link` 或 `world` 目标，可以先看：

```bash
ros2 topic echo --once /left_ee_current_pose_in_base
ros2 topic echo --once /left_ee_current_pose_in_world
```

```bash
ros2 topic pub --once /left_ee_target_pose geometry_msgs/msg/PoseStamped \
  "{header: {frame_id: left/arm_base}, pose: {position: {x: 0.25, y: 0.0, z: 0.25}, orientation: {w: 0.521333, x: 0.477719, y: 0.477719, z: 0.521326}}}"
```

观察：

- IK launch 终端是否打印 cuRobo 解算错误。
- `/left_ee_pose_ik_controller/status` 是否出现 `ik_succeeded`。
- Isaac 场景 5 里的左臂是否移动。
- 成功时 IK 终端会打印类似 `IK solution: left_joint1=...` 的关节解。

## 发送右臂目标位姿

默认启动参数已经包含右臂 IK，可以直接向 `/right_ee_target_pose` 发送右臂目标。右臂也建议使用你通过 FK 或实测验证过的自然姿态目标，不要先用 identity 姿态作为第一条测试。

下面是右臂命令模板，先把 `<tested_*>` 替换成右臂实测可解的目标位姿：

```bash
ros2 topic pub --once /right_ee_target_pose geometry_msgs/msg/PoseStamped \
  "{header: {frame_id: right/arm_base}, pose: {position: {x: <tested_x>, y: <tested_y>, z: <tested_z>}, orientation: {w: <tested_w>, x: <tested_xq>, y: <tested_yq>, z: <tested_zq>}}}"
```

## 当前约定

- cuRobo 使用 URDF 中的关节名，例如 `left/joint1`。
- Isaac 接收命令使用配置里的关节名，例如 `left_joint1`。
- `ee_pose_ik_controller` 内部负责做名称映射。
- 第一版只用 cuRobo 优化 `joint1` 到 `joint6`，cuRobo tool frame 仍是 `left/link6` / `right/link6`。
- 场景 USD 会给 `joint7` / `joint8` 加 linear drive，并把夹爪两指保持在安全开口，避免 TCP 因夹爪自由滑动而抖动。
- 对外 `ee_target_pose` 和 `ee_current_pose` 都表示动态 TCP：位置是 `center(link7, link8)`，姿态跟随 `link8`。
- 节点从 `/tf` 读取 `left_link6 -> left_link7/left_link8` 和右臂对应 TF，动态计算 TCP 相对 `link6` 的完整位姿偏移。
- `/left_ee_target_pose` 支持 `frame_id: left/arm_base`、`base_link`、`world`；`/right_ee_target_pose` 支持 `frame_id: right/arm_base`、`base_link`、`world`。
- `/left_ee_current_pose_in_arm_base` 和 `/right_ee_current_pose_in_arm_base` 由 cuRobo FK 和 Isaac `/tf` 共同计算出来，和 IK 使用同一套 kinematics/TCP 语义。
- `/left_ee_current_pose_in_arm_base` 的 `frame_id` 是 `left/arm_base`，`/right_ee_current_pose_in_arm_base` 的 `frame_id` 是 `right/arm_base`，两者数值不能直接互相比较。
- `/left_ee_current_pose_in_base` 和 `/right_ee_current_pose_in_base` 的 `frame_id` 是 `base_link`。它们会叠加 URDF 里的固定变换和当前 `lifting_joint`：`base_link -> lifting_link -> left/right arm_base -> link6`。
- `/left_ee_current_pose_in_world` 和 `/right_ee_current_pose_in_world` 的 `frame_id` 是 `world`。它们依赖 Isaac Sim 的 `ROS2 Publish Transform Tree` 在 `/tf` 中发布 `world -> base_link`。
- cuRobo 配置不把 `joint7` / `joint8` 作为 IK 自由度；这两个夹爪关节由 Isaac 场景里的 prismatic drive 保持在安全开口。
- 第一版调用 cuRobo `solve_pose(..., run_optimizer=True)` 做数值 IK 优化。`run_optimizer=False` 只适合检查 seed 是否已经满足目标，不适合作为真正的目标位姿求解。
- 第一版不做世界碰撞检查，不保证路径避开桌面、货架、另一只手臂或机器人本体。

## 常见问题

### `import curobo` 失败

进入 cuRobo 镜像容器检查：

```bash
python3 -c "import curobo; print('curobo ok')"
```

如果缺 Python 包，按报错补依赖。之前遇到过：

```bash
python3 -m pip install "setuptools_scm<8" wheel ninja tomli
```

如果看到 `'Configuration' object has no attribute 'scm'`，先用下面命令确认：

```bash
SETUPTOOLS_SCM_PRETEND_VERSION=0.0.0 python3 -c "import curobo; print('curobo ok')"
```

### `/left_ee_target_pose` 或 `/right_ee_target_pose` 没有 subscriber

说明 IK 节点没有启动。当前脚本的 `--launch-control` 默认会补上：

```bash
start_ee_pose_ik:=true ik_side:=both
```

如果你手动追加了 `start_ee_pose_ik:=false`，则不会启动左右 IK 节点。

### `/isaac_joint_commands` 或内部平滑 topic 没有 subscriber

说明插值节点、Isaac 侧 Action Graph 或场景 Play 状态不对。检查：

- 是否打开的是 `assets/5.mobile_aloha_in_market_ros_camera.usda`。
- 是否点击 Play。
- `isaacsim.ros2.bridge` 是否启用。
- Docker 控制 launch 里 `joint_command_interpolator` 是否启动。

### IK 成功但机械臂不动

检查是否有其他节点同时控制同一条机械臂，例如 `arm_joint_command_demo`。同一批关节不要同时被多个节点发布命令。

### `_in_world` 末端位姿没有输出

检查 Isaac Sim 里是否添加并启用了 `ROS2 Publish Transform Tree`，并确认 `/tf` 里有：

```text
frame_id: world
child_frame_id: base_link
```

### TCP 末端位姿没有输出或 IK 报等待 TCP TF

检查 `/tf` 里是否有夹爪两指相对 `link6` 的变换：

```text
frame_id: left_link6
child_frame_id: left_link7
child_frame_id: left_link8
```

右臂对应为 `right_link6 -> right_link7/right_link8`。
