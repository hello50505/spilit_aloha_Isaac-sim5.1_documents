# 用 MoveIt2 + TRAC-IK 替换 cuRobo IK 后端

本文是 `15_debug_IK.md` 之后的第二步：在已有控制链路上把 IK
求解器从 cuRobo 换成 MoveIt2 提供的 IK service，重点是绕开
cuRobo 在 6-DOF 边界处反复失败的问题。

## 为什么换

`15_debug_IK.md` 给的 cuRobo 调参方案（放宽 `rotation_tolerance`、
增大 `num_seeds`、调 `default_joint_position`、加 fallback）做完
仍然在边界处大量失败。再加上 SpaceMouse 遥操作时目标会持续
被推到工作空间外缘，体感是 IK 链路"动不动就死一阵"。

替换 IK 后端之后预期能解决的：

- 6-DOF Piper 在边界处的可达解算成功率显著上升（TRAC-IK 在
  6-DOF 上对 KDL 和 cuRobo 都更稳）。
- 失败时返回的 error code 更细，可以分清"目标真不可达"还是
  "求解超时"。
- 之后接 self-collision、世界碰撞、`MoveIt Servo`、RViz 交互式
  marker 都顺手；先迈第一步把 IK 切过去。

替换 IK 后端**不能**解决的（这条要先讲清楚，避免迁完发现没用）：

- SpaceMouse 旋转和末端"对不齐"的感觉。这是 SpaceMouse → 目
  标姿态映射层的事，发生在 `mobile_aloha_debug_teleop.py` 内
  部，与 IK backend 没关系。继续按 `8_debug_teleop.md` 里的
  `rotation_frame: body|world` 切换 + `space_*_sign` 调；这一
  层调通后再看 IK。
- `lifting_link/right_link*` 之类的幽灵碰撞（参见
  `14_debug_ghost_collision.md`），属于场景层 collider 的问题。
- cuRobo 现成的 motion gen / batched IK，迁到 MoveIt 后这两点
  会消失，用 OMPL/MoveIt2 servo 顶替；但本文只换 IK 服务，规
  划暂时不接。

## 接法选型

MoveIt2 集成有三种粒度：

| 选型 | 说明 | 工作量 |
| --- | --- | --- |
| A. 仅 IK service | 起 `move_group`，**只调用** `/compute_ik`，关节命令仍由现有节点直接发 `/isaac_joint_commands` | 小 |
| B. IK + Servo | 用 `MoveIt Servo` 做 6-DOF 增量伺服，输出 `JointTrajectory`，需要 `JointTrajectory` → Isaac 的 bridge | 中 |
| C. 完整 MoveIt | RViz 交互式目标 + OMPL 路径规划 + `JointTrajectory` 控制器 | 大 |

本文走 **A**。原因：

- 现有 `ee_pose_ik_controller.py` 已经把"PoseStamped → 关节命
  令"的全流程抽好了：解 IK、内部 max-step 平滑、输出
  `/isaac_joint_commands`。直接给它加一个 `backend: moveit` 分
  支调 `/compute_ik` 即可，原 cuRobo 路径保留为可选 fallback。
- 不动 `joint_command_interpolator`、`mobile_aloha_debug_teleop`、
  `right_ee_target_marker_publisher` 这些节点；READY 表里 `[3]`
  `[4]` 的语义不变。
- Docker 多装 `ros-humble-moveit*` 走 apt；TRAC-IK 走源码（Humble
  没有 apt 二进制），克隆到 `ros2_ws/src/trac_ik` 跟其他包一起
  `colcon build`，不引入新的工作区。

将来想升 B/C 时再单独写 `17_*.md`，这次专注 A。

## 整体改动面

```text
mobile_aloha_sim/split_aloha_mid_360/urdf/                         # 现有 URDF, 不动
dev_ws/ros2_ws/src/mobile_aloha_moveit_config/                     # 新增, MoveIt 配置包
  package.xml
  CMakeLists.txt
  config/
    split_aloha.srdf
    kinematics.yaml
    joint_limits.yaml
    moveit_controllers.yaml          # 仅占位, MoveIt Servo/规划用; 本步不连
  launch/
    move_group.launch.py             # 仅启 move_group + IK 后端, 不连 controller_manager
dev_ws/ros2_ws/src/trac_ik/                                        # 新增, 源码 (bitbucket.org/traclabs/trac_ik, rolling 分支)
                                                                   # 通过 colcon 编译, Humble 没有 apt 二进制
dev_ws/ros2_ws/src/mobile_aloha_isaac_control/
  config/
    mobile_aloha_joints.yaml         # 加 backend 切换
  mobile_aloha_isaac_control/
    ee_pose_ik_controller.py         # 加 backend == "moveit" 的分支
  launch/
    mobile_aloha_control.launch.py   # 启 move_group 子 launch (可选)
dev_ws/scripts/
  run_ros2_humble_docker.sh          # 默认镜像改成 :moveit (apt 装 moveit, 源码编译 trac_ik)
```

## 第一步：扩 Docker 镜像 + 源码编译 TRAC-IK

`mobile_aloha_ros2_humble:curobo` 已经装好了 cuRobo + ROS2 humble。
直接在它基础上叠加 MoveIt 和 TRAC-IK，避免重复构建 cuRobo。

> **关于 TRAC-IK**：`trac_ik` 在 ROS 2 buildfarm 上只发布到
> Rolling / Jazzy / Kilted，**没有 Humble apt 二进制包**
> （`E: Unable to locate package ros-humble-trac-ik-kinematics-plugin`）。
> 走源码编译。**仓库选 TRACLabs 上游**
> [`bitbucket.org/traclabs/trac_ik`](https://bitbucket.org/traclabs/trac_ik)，
> **checkout 到 tag `2.0.1`**（4 个包都是 ROS 2，
> `ament_cmake` + `rclcpp`）。
>
> ⚠️ **不能直接用 `rolling` 分支**：从 v2.1.0 开始上游统一
> 切到 ROS 2 新风格头文件（`urdf/model.hpp`、
> `moveit/kinematics_base/kinematics_base.hpp` 等），但
> Humble 的 `urdf` / `moveit_core` 还是 `.h`，会编不过：
> `fatal error: urdf/model.hpp: No such file or directory`。
> 2.0.2 的 plugin 也已经切到 `moveit/.../*.hpp`，所以
> Humble 上的最新可用 tag 是 **2.0.1**（`urdf/model.h` +
> `moveit/.../*.h`，全 Humble 兼容）。
>
> ⚠️ 也**不要**用 [`aprotyas/trac_ik`](https://github.com/aprotyas/trac_ik) `ros2`
> 分支：只移植了 `trac_ik_lib` 和 `trac_ik_examples`，
> `trac_ik_kinematics_plugin/` 目录里有 `COLCON_IGNORE`，
> `package.xml` 还是 catkin 的，colcon 会直接报
> `ignoring unknown package 'trac_ik_kinematics_plugin'`。
>
> 系统依赖：`libnlopt-dev`、`generate_parameter_library`、
> `tf2_kdl`、`eigen3_cmake_module`、`kdl_parser` —— `rosdep`
> 会全自动拉。

启动一次容器：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
ISAAC_ROS2_IMAGE=mobile_aloha_ros2_humble:curobo \
  ./scripts/run_ros2_humble_docker.sh
```

容器内先装 MoveIt + 系统依赖（注意：apt 列表里 **没有**
`ros-humble-trac-ik-kinematics-plugin`）：

```bash
apt update
DEBIAN_FRONTEND=noninteractive apt install -y \
  ros-humble-moveit \
  ros-humble-moveit-resources \
  ros-humble-moveit-servo \
  ros-humble-xacro \
  libnlopt-dev libnlopt-cxx-dev \
  git
```

接着克隆 TRAC-IK 源码到 `ros2_ws/src/`（这条路径是 host bind
mount，下次进容器还在），**checkout 到 2.0.1 tag**：

```bash
cd /workspace/dev_ws/ros2_ws/src
if [ ! -d trac_ik ]; then
  git clone https://bitbucket.org/traclabs/trac_ik.git
  ( cd trac_ik && git checkout 2.0.1 )
fi
```

如果之前误克隆了别的版本（`aprotyas/trac_ik`、bitbucket
`rolling` 分支等），先清掉旧 clone 和 stale 产物再换源：

```bash
cd /workspace/dev_ws/ros2_ws
rm -rf src/trac_ik \
       build/trac_ik_lib build/trac_ik_examples build/trac_ik_kinematics_plugin \
       install/trac_ik_lib install/trac_ik_examples install/trac_ik_kinematics_plugin \
       log
cd src
git clone https://bitbucket.org/traclabs/trac_ik.git
( cd trac_ik && git checkout 2.0.1 )
```

回到工作区根，跑 rosdep + colcon：

```bash
cd /workspace/dev_ws/ros2_ws
source /opt/ros/humble/setup.bash
rosdep update
rosdep install --from-paths src/trac_ik --ignore-src -r -y

colcon build --symlink-install --packages-select \
  trac_ik_lib trac_ik_kinematics_plugin trac_ik_examples
source install/setup.bash

ros2 pkg list | grep -E "moveit|trac" | head
```

至少应看到：

```text
moveit_core
moveit_kinematics
moveit_msgs
moveit_ros_move_group
moveit_ros_planning
moveit_ros_planning_interface
moveit_servo
trac_ik_kinematics_plugin
trac_ik_lib
```

`trac_ik_kinematics_plugin` 出现就说明 MoveIt 之后能通过
`pluginlib` 找到 `TRAC_IKKinematicsPlugin`，第三步那份
`kinematics.yaml` 才能加载得起来。

宿主机另开终端保存当前容器：

```bash
docker commit isaac_ros2_humble mobile_aloha_ros2_humble:moveit
```

> **要注意 commit 的边界**：apt 装的 MoveIt / nlopt 在镜像 rootfs，
> 会被 commit 进去。`trac_ik` 的源码和 `install/` 落在
> `/workspace/dev_ws/ros2_ws`，那是 host bind mount，**不会**进
> 镜像；但下次启动同一个仓库时它还在 host 路径里，
> `run_ros2_humble_docker.sh` 里 `if [ -f install/setup.bash ];
> then source install/setup.bash; fi` 会自动 source 进来，所以
> 不用再编。换 host / 别人 clone 仓库时，需要重新跑一遍上面
> 那段 `git clone` + `colcon build`。

之后启动时改用：

```bash
ISAAC_ROS2_IMAGE=mobile_aloha_ros2_humble:moveit \
  ./scripts/run_ros2_humble_docker.sh --build --launch-control
```

`run_ros2_humble_docker.sh` 默认镜像也可以一并改成
`mobile_aloha_ros2_humble:moveit`，这样脚本就不用每次显式传
`ISAAC_ROS2_IMAGE`。

## 第二步：为机械臂写 SRDF

MoveIt 必须有 SRDF（语义机器人描述）。当前我们只关心两条手
臂，所以新建一个最小包 `mobile_aloha_moveit_config`，里面给两
条手臂各定义一个 planning group，关节列表和 `mobile_aloha_joints.yaml`
里 `solver_joint_names` 完全对齐。

`dev_ws/ros2_ws/src/mobile_aloha_moveit_config/package.xml`：

```xml
<?xml version="1.0"?>
<package format="3">
  <name>mobile_aloha_moveit_config</name>
  <version>0.0.1</version>
  <description>MoveIt2 config for Mobile ALOHA Piper arms.</description>
  <maintainer email="xiangpc@example.com">xiangpc</maintainer>
  <license>BSD</license>

  <buildtool_depend>ament_cmake</buildtool_depend>

  <exec_depend>moveit_ros_move_group</exec_depend>
  <exec_depend>moveit_kinematics</exec_depend>
  <exec_depend>trac_ik_kinematics_plugin</exec_depend>
  <exec_depend>xacro</exec_depend>
  <exec_depend>robot_state_publisher</exec_depend>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

`CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.16)
project(mobile_aloha_moveit_config)
find_package(ament_cmake REQUIRED)
install(DIRECTORY config launch DESTINATION share/${PROJECT_NAME})
ament_package()
```

`config/split_aloha.srdf` 的关键骨架（先写最小可解 IK 版，
self-collision 这次不打开，避免引入 14 号文档里的幽灵碰撞）：

```xml
<?xml version="1.0"?>
<!-- robot name MUST match URDF root name (split_aloha_rslidar_with_piper);
     mismatching names trigger urdfdom's
       "Semantic description is not specified for the same robot as the URDF"
     red error during move_group startup.  -->
<robot name="split_aloha_rslidar_with_piper">
  <group name="left_arm">
    <chain base_link="left/arm_base" tip_link="left/link6"/>
  </group>
  <group name="right_arm">
    <chain base_link="right/arm_base" tip_link="right/link6"/>
  </group>

  <group_state name="left_home" group="left_arm">
    <joint name="left/joint1" value="0.0"/>
    <joint name="left/joint2" value="0.4"/>
    <joint name="left/joint3" value="-0.6"/>
    <joint name="left/joint4" value="0.0"/>
    <joint name="left/joint5" value="0.5"/>
    <joint name="left/joint6" value="0.0"/>
  </group_state>
  <group_state name="right_home" group="right_arm">
    <joint name="right/joint1" value="0.0"/>
    <joint name="right/joint2" value="0.4"/>
    <joint name="right/joint3" value="-0.6"/>
    <joint name="right/joint4" value="0.0"/>
    <joint name="right/joint5" value="0.5"/>
    <joint name="right/joint6" value="0.0"/>
  </group_state>

  <!-- 必须用 floating, 不能用 fixed.
       Isaac Sim 把这台车 spawn 在 world 系下某个非原点位置, 例如
       (-3.28, -0.28, 0.55). SRDF 如果写 fixed, MoveIt 内部就把
       base_link 钉死在 world 原点, 它做的 world->arm_base FK
       跟 ee_pose_ik_controller._arm_base_pose_to_world 用真实
       /tf 算的 world target 差几米, 同一个 IK 请求会被 MoveIt
       解释成"请把 EE 送到机器人 3 米外", 直接 NO_IK_SOLUTION.
       改 floating 之后, 每次 /compute_ik 请求里我们用
       RobotState.multi_dof_joint_state 把真实 world->base_link
       喂给 MoveIt, 两边 FK 才能对齐. -->
  <virtual_joint name="virtual_world" type="floating"
                 parent_frame="world" child_link="base_link"/>

  <disable_collisions link1="lifting_link" link2="box_link" reason="Adjacent"/>
  <disable_collisions link1="left/link1"  link2="left/arm_base"  reason="Adjacent"/>
  <disable_collisions link1="right/link1" link2="right/arm_base" reason="Adjacent"/>
</robot>
```

注意几点：

- `chain` 的 `base_link/tip_link` 必须**和 cuRobo `mobile_aloha_right.yml`
  里的 `base_link` / `tool_frames` 完全一致**：`left/arm_base` →
  `left/link6`，右臂同理。这样新旧后端语义可以直接对齐。
- `virtual_joint` 把 SRDF 的世界根节点固定到 URDF 的 `base_link`，
  对应的 frame 名是 `world`。这一行让 MoveIt 内部状态空间能
  正确读到 `world -> base_link` 的 TF。
- `disable_collisions` 这次只关掉最近邻几对，避免 SRDF 默认
  规则把 `link7/link8` 也关进去（影响后续抓瓶碰撞）。完整的
  Adjacent 集合可以等以后跑 `moveit_setup_assistant` 自动生
  成，本步不打开 self-collision check 也不会用到。
- 分组用 `left_arm` / `right_arm`。后面 `kinematics.yaml` 按这
  两个组各配一份 TRAC-IK。

如果你愿意一开始就用 GUI，可以在宿主机起一个带显示的
ROS2 Humble Docker 跑：

```bash
ros2 run moveit_setup_assistant moveit_setup_assistant
```

然后选择 URDF：

```text
mobile_aloha_sim/split_aloha_mid_360/urdf/split_aloha_rslidar_with_piper.urdf
```

按向导建 `left_arm` / `right_arm` 两个组，`Self-Collisions` 让
它自动生成（本步生成完后我们手动改回来禁用，避免影响调试），
最后导出到 `dev_ws/ros2_ws/src/mobile_aloha_moveit_config/`。
GUI 跑一次以后就不用再开了。

## 第三步：`kinematics.yaml` 用 TRAC-IK

`config/kinematics.yaml`：

```yaml
left_arm:
  kinematics_solver: trac_ik_kinematics_plugin/TRAC_IKKinematicsPlugin
  kinematics_solver_search_resolution: 0.005
  kinematics_solver_timeout: 0.05
  kinematics_solver_attempts: 3
  solve_type: Distance
  position_only_ik: false
  epsilon: 1e-5

right_arm:
  kinematics_solver: trac_ik_kinematics_plugin/TRAC_IKKinematicsPlugin
  kinematics_solver_search_resolution: 0.005
  kinematics_solver_timeout: 0.05
  kinematics_solver_attempts: 3
  solve_type: Distance
  position_only_ik: false
  epsilon: 1e-5
```

参数说明：

- `solve_type: Distance`：在等价解里选"距离当前关节最近"的那
  一个，遥操作连续性最好。其他选项 `Speed` 求解最快但解之间
  跳变较大，`Manipulation1/2` 偏向规避奇异。
- `kinematics_solver_timeout: 0.05`：单次 IK 50 ms 上限。TRAC-IK
  实际平均 1–5 ms，这个值是上界保险，避免遥操作堵塞。
- `position_only_ik: false`：仍要解姿态。如果以后想做"位置
  fallback"（参考 15 号文档方案 D），可以单独再起一个 group
  `right_arm_pos_only`，把这个开关打开。
- `epsilon: 1e-5`：TRAC-IK 内部收敛阈值，对应 `15_debug_IK.md`
  里 cuRobo 的 `position_tolerance_m`，但单位是无量纲，1e-5
  在 6-DOF Piper 上经验值比 cuRobo 5 mm 容忍度更紧但仍稳定。

## 第四步：`joint_limits.yaml`

直接复用 URDF 里的 limits，加一行加速度上限给 MoveIt（URDF 不
带）。具体值不重要，本文 A 模式下 IK 不规划路径，给个保守值
就行：

```yaml
joint_limits:
  left/joint1:  { has_velocity_limits: true, max_velocity: 2.0, has_acceleration_limits: true, max_acceleration: 5.0 }
  left/joint2:  { has_velocity_limits: true, max_velocity: 2.0, has_acceleration_limits: true, max_acceleration: 5.0 }
  left/joint3:  { has_velocity_limits: true, max_velocity: 2.0, has_acceleration_limits: true, max_acceleration: 5.0 }
  left/joint4:  { has_velocity_limits: true, max_velocity: 3.0, has_acceleration_limits: true, max_acceleration: 8.0 }
  left/joint5:  { has_velocity_limits: true, max_velocity: 3.0, has_acceleration_limits: true, max_acceleration: 8.0 }
  left/joint6:  { has_velocity_limits: true, max_velocity: 3.0, has_acceleration_limits: true, max_acceleration: 8.0 }
  right/joint1: { has_velocity_limits: true, max_velocity: 2.0, has_acceleration_limits: true, max_acceleration: 5.0 }
  right/joint2: { has_velocity_limits: true, max_velocity: 2.0, has_acceleration_limits: true, max_acceleration: 5.0 }
  right/joint3: { has_velocity_limits: true, max_velocity: 2.0, has_acceleration_limits: true, max_acceleration: 5.0 }
  right/joint4: { has_velocity_limits: true, max_velocity: 3.0, has_acceleration_limits: true, max_acceleration: 8.0 }
  right/joint5: { has_velocity_limits: true, max_velocity: 3.0, has_acceleration_limits: true, max_acceleration: 8.0 }
  right/joint6: { has_velocity_limits: true, max_velocity: 3.0, has_acceleration_limits: true, max_acceleration: 8.0 }
```

## 第五步：只启 `move_group`，不连 controller manager

`launch/move_group.launch.py` 的最终落地版（已踩过两个上游坑，
注释写在代码里）：

- `kinematics.yaml` / `joint_limits.yaml` 必须 `yaml.safe_load`
  成 dict 后塞进 parameter，**不能直接传字符串路径**——MoveIt
  的 `KinematicsLoader` 是从 `robot_description_kinematics.<group>.kinematics_solver`
  这个 nested 参数空间里读，传字符串读不到。
- `URDF` 里的 `package://...` 引用要先 inline 替换成绝对路径，
  跟 `ee_pose_ik_controller._sanitize_urdf_package_paths` 同
  口径，否则 `move_group` 解析 mesh 时会报红字（mesh 缺失非
  致命，但 URDF 里 `package://piper_description/...` 这种路径
  在容器里没有 ROS pkg index 解析得到）。
- ⚠️ **不要**设 `capabilities: "move_group/MoveGroupComputeIKService"`：
  这个类名是网上教程里的常见错误，Humble 实际类名是
  `move_group/MoveGroupKinematicsService`。设错之后 pluginlib
  会抛 `Exception while loading move_group capability ... does
  not exist` 然后**回退到默认全套** capability（OMPL/CHOMP/Pilz
  /MoveAction/Cartesian 全加载），跟我们想要的"轻量 IK only"
  完全相反。最安全的写法是**完全不设这个参数**，依赖 Humble
  默认 capability 集（已经包含 `KinematicsService`），同时
  `allow_trajectory_execution: false` 抑制 controller 调用。
- ⚠️ `move_group` 默认订阅 `/joint_states` 喂 `current_state_monitor`，
  但 Isaac Sim 那边发的是下划线名（`left_joint1`），URDF/SRDF
  里是斜线名（`left/joint1`），monitor 找不到关节会刷
  `Joint 'left_joint1' not found in model`。我们 `compute_ik`
  请求里每次都带完整 seed，monitor 状态对 IK 没意义，最干净
  的处理是把 `joint_states` remap 到一个没人发的 dead topic。

```python
from pathlib import Path

import yaml
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch_ros.actions import Node


_URDF_PACKAGE_REPLACEMENTS = {
    "package://split_aloha_mid_360": "/workspace/isaac-sim/mobile_aloha_sim/split_aloha_mid_360",
    "package://piper_description": "/workspace/isaac-sim/mobile_aloha_sim/piper_description",
}


def _sanitize_urdf(text: str) -> str:
    for src, dst in _URDF_PACKAGE_REPLACEMENTS.items():
        text = text.replace(src, dst)
    return text


def _load_yaml(path: Path) -> dict:
    with path.open("r", encoding="utf-8") as stream:
        return yaml.safe_load(stream) or {}


def generate_launch_description() -> LaunchDescription:
    moveit_share = Path(get_package_share_directory("mobile_aloha_moveit_config"))
    urdf_path = Path(
        "/workspace/isaac-sim/mobile_aloha_sim/split_aloha_mid_360/urdf/"
        "split_aloha_rslidar_with_piper.urdf"
    )
    srdf_path = moveit_share / "config" / "split_aloha.srdf"
    kinematics_path = moveit_share / "config" / "kinematics.yaml"
    joint_limits_path = moveit_share / "config" / "joint_limits.yaml"

    robot_description = {"robot_description": _sanitize_urdf(urdf_path.read_text())}
    robot_description_semantic = {
        "robot_description_semantic": srdf_path.read_text()
    }
    kinematics_yaml = _load_yaml(kinematics_path)
    joint_limits_yaml = _load_yaml(joint_limits_path)

    return LaunchDescription(
        [
            Node(
                package="moveit_ros_move_group",
                executable="move_group",
                name="move_group",
                output="screen",
                parameters=[
                    robot_description,
                    robot_description_semantic,
                    {"robot_description_kinematics": kinematics_yaml},
                    {"robot_description_planning": joint_limits_yaml},
                    {"publish_robot_description_semantic": True},
                    {"publish_robot_description": False},
                    {"allow_trajectory_execution": False},
                    {"use_sim_time": False},
                ],
                remappings=[
                    ("joint_states", "/move_group_internal_joint_states_unused"),
                ],
            ),
        ]
    )
```

关键：

- `allow_trajectory_execution: False`：不连 `controller_manager`
  和任何 `JointTrajectoryController`。`move_group` 启动后只暴
  露 `/compute_ik`、`/get_planning_scene` 等服务，不会去找
  controller，避免和现有 `joint_command_interpolator` 抢
  `/isaac_joint_commands`。
- 不设 `capabilities`：见上文 ⚠️。Humble 默认集已经包含
  `KinematicsService`，`/compute_ik` 自然就有。
- `joint_states` remap：见上文 ⚠️。
- `urdf_path` 走绝对路径，复用 cuRobo `right.yml` 里相同的 URDF
  文件，避免两份 URDF drift。

## 第六步：让 `ee_pose_ik_controller.py` 多一个 `backend: moveit` 分支

`mobile_aloha_joints.yaml`：

```yaml
ee_pose_control:
  publish_rate_hz: 50.0
  max_joint_step_rad: 0.06
  position_tolerance_m: 0.005
  rotation_tolerance_rad: 0.20
  backend: moveit
  moveit:
    compute_ik_service: /compute_ik
    timeout_s: 0.05
    avoid_collisions: false
  sides:
    left:
      moveit_group: left_arm
      moveit_tip_link: left/link6
      ...
    right:
      moveit_group: right_arm
      moveit_tip_link: right/link6
      ...
```

`ee_pose_ik_controller.py` 的修改要点（伪代码）：

```python
self.backend = str(self.get_parameter("backend").value or ee_config["backend"])
if self.backend not in {"curobo", "moveit"}:
    raise ValueError("backend must be one of: curobo, moveit")

self._moveit_client: Optional[Client] = None
if self.backend == "moveit":
    from moveit_msgs.srv import GetPositionIK
    self._moveit_client = self.create_client(
        GetPositionIK,
        ee_config["moveit"]["compute_ik_service"],
    )
    if not self._moveit_client.wait_for_service(timeout_sec=10.0):
        raise RuntimeError("Timed out waiting for /compute_ik")
```

`_on_target_pose` 里把 `self._solve_curobo(msg)` 替换成调度
分支：

```python
def _on_target_pose(self, msg: PoseStamped) -> None:
    started_at = time.perf_counter()
    try:
        if self.backend == "curobo":
            self.target_positions = self._solve_curobo(msg)
        else:
            self.target_positions = self._solve_moveit(msg)
    except Exception as exc:
        ...
```

`_solve_moveit` 主体（要点写完整）：

```python
def _solve_moveit(self, msg: PoseStamped) -> List[float]:
    from moveit_msgs.msg import PositionIKRequest, RobotState
    from moveit_msgs.srv import GetPositionIK
    from sensor_msgs.msg import JointState

    # 先按老路径算出 (left|right)/arm_base 系下的 link6 目标
    target_arm_base = self._target_link6_pose_in_arm_base(msg)
    # 然后正向变换到 world 系（绕开 MoveIt 的 tf2 查询，见下面解释）
    target_world = self._arm_base_pose_to_world(
        target_arm_base["position"], target_arm_base["quaternion"]
    )

    request = GetPositionIK.Request()
    ik_req = PositionIKRequest()
    ik_req.group_name = self.side_config["moveit_group"]
    ik_req.timeout = Duration(
        seconds=self.config["ee_pose_control"]["moveit"]["timeout_s"]
    ).to_msg()
    ik_req.avoid_collisions = bool(
        self.config["ee_pose_control"]["moveit"]["avoid_collisions"]
    )
    ik_req.ik_link_name = self.side_config["moveit_tip_link"]

    # seed 必须包含整条 base_link → arm_base 链路上的可动关节,
    # 这里就是手臂 6 个关节 + lifting_joint:
    seed = JointState()
    seed.name = list(self.solver_joint_names)
    seed.position = [
        self.current_positions.get(cmd, fallback)
        for cmd, fallback in zip(self.command_joint_names, self.command_positions)
    ]
    if self.lifting_joint_name in self.current_positions:
        seed.name.append(self.lifting_joint_name)
        seed.position.append(self.current_positions[self.lifting_joint_name])
    ik_req.robot_state = RobotState(joint_state=seed, is_diff=True)

    pose_stamped = PoseStamped()
    pose_stamped.header.frame_id = self.world_frame  # 必须是 SRDF model_frame
    pose_stamped.pose.position.x = float(target_world["position"][0])
    pose_stamped.pose.position.y = float(target_world["position"][1])
    pose_stamped.pose.position.z = float(target_world["position"][2])
    quat = target_world["quaternion"]   # [w,x,y,z]
    pose_stamped.pose.orientation.w = float(quat[0])
    pose_stamped.pose.orientation.x = float(quat[1])
    pose_stamped.pose.orientation.y = float(quat[2])
    pose_stamped.pose.orientation.z = float(quat[3])
    ik_req.pose_stamped = pose_stamped

    request.ik_request = ik_req
    future = self._moveit_client.call_async(request)
    rclpy.spin_until_future_complete(self, future, timeout_sec=0.1)
    if not future.done():
        raise RuntimeError("compute_ik service call timed out")
    response = future.result()
    if response is None:
        raise RuntimeError("compute_ik returned None")
    err = response.error_code.val
    if err != response.error_code.SUCCESS:
        raise RuntimeError(f"compute_ik failed, error_code={err}")

    name_to_pos = dict(
        zip(response.solution.joint_state.name,
            response.solution.joint_state.position)
    )
    positions = [name_to_pos.get(name, 0.0) for name in self.solver_joint_names]
    return positions[: len(self.command_joint_names)]
```

注意几点：

- **PoseStamped 必须发在 `world` 系而不是 `(left|right)/arm_base`
  系**。原因和 Isaac Sim 的 TF 命名约定有关：URDF 里链接名是
  `right/arm_base`、`right/link6` 这种带斜线的，但 Isaac 把 URDF
  导成 USD 后再 publish `/tf`，会把斜线全部替换成下划线，于是
  topic 上看到的是 `right_arm_base`、`right_link6`。MoveIt 的
  `MoveGroupKinematicsService::performTransform` 会拿
  `pose_msg.header.frame_id` 去查它自己的 tf2 buffer，找
  `right/arm_base` 永远找不到，只能拿到 `right_arm_base`，结果
  IK service 返回 `error_code=-21 (FRAME_TRANSFORM_FAILURE)`。
  规避办法是把 `frame_id` 设成 SRDF 的 model_frame（也就是
  `world`）：`performTransform` 在 `frame_id == target_frame`
  时直接 short-circuit，根本不会触发 tf2 查询。我们自己用
  `_arm_base_pose_to_world` 把 arm_base 系下的 link6 目标
  正向变换到 world 系，这一段链路 (`world → base_link →
  box_link → lifting_link → *_arm_base`) 在 URDF 里所有
  `<origin>` 的 `rpy` 都是 0，所以是纯平移、容易写。
- **seed 里要带 `lifting_joint`**。MoveIt 在拿到 world 系下的
  pose 之后，还要再用 *seed* 的 FK 把它转到 solver 的 chain base
  (`right/arm_base`)，这条链路上唯一可动的就是 `lifting_joint`。
  如果不喂这个关节，MoveIt 默认它=0，算出来的 `world →
  arm_base` 跟我们用真实 lifting 算出来的 world 目标不一致，
  IK 就整体跑偏。`is_diff=True` 让 seed 仅覆盖手臂这 6+1 个
  关节，其他关节继续用 RobotModel 的默认值，刚好就是我们
  remap 掉 `joint_states` 之后的状态。
- **SRDF 必须 `virtual_joint type="floating"`, IK 请求里再用
  `multi_dof_joint_state` 把真实 `world → base_link` 喂回去**.
  Isaac Sim 把整台车 spawn 在 world 系一个非原点位置 (例如
  `(-3.28, -0.28, 0.55)`), `_arm_base_pose_to_world` 用真实
  `world_to_base` 算 world target, 如果 SRDF 还写 fixed, MoveIt
  内部默认 base 在原点, 内部 FK 算的 `world → arm_base` 跟我们
  用的差几米, 直接 `NO_IK_SOLUTION`. 喂法:

  ```python
  from geometry_msgs.msg import Quaternion as QuaternionMsg
  from geometry_msgs.msg import Transform, Vector3
  from sensor_msgs.msg import MultiDOFJointState

  multi_dof = MultiDOFJointState()
  multi_dof.header.stamp = self.get_clock().now().to_msg()
  multi_dof.header.frame_id = self.world_frame  # "world"
  multi_dof.joint_names = ["virtual_world"]      # 与 SRDF 名字一致
  multi_dof.transforms = [Transform(
      translation=Vector3(x=wtb_pos[0], y=wtb_pos[1], z=wtb_pos[2]),
      rotation=QuaternionMsg(w=wtb_quat[0], x=wtb_quat[1],
                             y=wtb_quat[2], z=wtb_quat[3]),
  )]
  ik_req.robot_state.multi_dof_joint_state = multi_dof
  ```

  这样 MoveIt 内部 `world → base_link` 等于真实 TF, 跟我们的正
  向变换对齐, 不管底盘停在哪都能解.
- `seed` 来自当前 `/joint_states`，TRAC-IK 默认会把 seed 作为
  起点搜索，连续性比 cuRobo 还好。
- `error_code.val == SUCCESS` 是唯一成功标识。`NO_IK_SOLUTION`、
  `TIMED_OUT`、`GOAL_CONSTRAINTS_VIOLATED` 都直接抛 RuntimeError，
  日志里能看清原因，比 cuRobo 那行 "optimizer failed" 信息量
  大得多。
- `rclpy.spin_until_future_complete` 在 `_on_target_pose` 这种
  subscription callback 里**不能直接用默认 executor**，否则
  会触发 reentrancy 报错。需要单独用一个
  `MutuallyExclusiveCallbackGroup`，或者在 `Node.__init__` 里
  把 IK 调用单独放到一个 `ReentrantCallbackGroup` + 多线程
  executor。最干净的写法是在 main 里：

  ```python
  executor = MultiThreadedExecutor(num_threads=2)
  executor.add_node(node)
  executor.spin()
  ```

  并且把 subscription 和 service client 各放一个不同的
  callback group。如果偷懒先单线程跑，会观察到 `compute_ik`
  迟迟不返回；这时换 multi-threaded executor。

## 第七步：launch 集成

`mobile_aloha_control.launch.py` 加一个开关 `start_move_group`，
默认随 `start_ee_pose_ik` 一起开：

```python
DeclareLaunchArgument("start_move_group", default_value="true"),
DeclareLaunchArgument("ik_backend", default_value="moveit"),
...
IncludeLaunchDescription(
    PythonLaunchDescriptionSource([
        PathJoinSubstitution([
            FindPackageShare("mobile_aloha_moveit_config"),
            "launch",
            "move_group.launch.py",
        ]),
    ]),
    condition=IfCondition(LaunchConfiguration("start_move_group")),
),
```

`ee_pose_ik_controller` 节点参数里把 backend 透传：

```python
parameters=[
    {"backend": LaunchConfiguration("ik_backend")},
    ...
],
```

这样默认 launch 就会同时起 `move_group` 和两个 IK 节点，
backend = `moveit`。

需要回到 cuRobo 时只要：

```bash
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  start_move_group:=false ik_backend:=curobo
```

## 第八步：在控制 launch 里给 READY 表加一行

`8_debug_teleop.md` 里 READY 表当前是 `[0]–[5]`。加 `move_group`
之后建议把它的 READY 也算进来，避免 IK 节点先起来连不到
`/compute_ik`。最简单的处理：在 `ee_pose_ik_controller` 内
`backend == "moveit"` 时，`_log_ready_if_needed` 里加一条
`self._moveit_client.service_is_ready()` 的检查。`move_group`
本身不打 READY 行，让 IK 节点的 `[3] [4]` READY 自然包含
service 可用。

## 验证流程

整个迁完之后按下面顺序验证。

1. 启 Isaac Sim：

   ```bash
   cd /home/xiangpc/dataset/isaac-sim
   ./dev_ws/scripts/run_isaac_mobile_aloha.sh
   ```

   打开 `assets/6_aloha_in_blue_grid_ROS.usda`，Play。

2. 容器里编译并起 launch：

   ```bash
   cd /home/xiangpc/dataset/isaac-sim/dev_ws
   ISAAC_ROS2_IMAGE=mobile_aloha_ros2_humble:moveit \
     ./scripts/run_ros2_humble_docker.sh --build --launch-control
   ```

3. 等所有进程 READY。`move_group` 起来之前不会有红字，注意它
   的 INFO 行里出现 `Loading planning_request_adapters` 即可
   忽略。`[3]` `[4]` 的 READY 行说明 IK 节点已经成功握手
   `/compute_ik`。

4. 打开第二个容器 shell 测试服务存在：

   ```bash
   ros2 service list | grep compute_ik
   ros2 service type /compute_ik
   ```

   应输出 `moveit_msgs/srv/GetPositionIK`。

5. 单步基线复测。`/right_ee_target_pose` 这一层的 `frame_id`
   接受 `world` / `base_link` / `right/arm_base` / `right/tcp`
   四种（见 `_target_tcp_pose_in_arm_base`），下面这条是从
   `right/arm_base` 系发一个明确可达目标：

   ```bash
   ros2 topic pub --once /right_ee_target_pose geometry_msgs/msg/PoseStamped \
     "{header: {frame_id: right/arm_base},
       pose: {position: {x: 0.30, y: 0.0, z: 0.20},
              orientation: {w: 0.521, x: 0.478, y: 0.478, z: 0.521}}}"
   ```

   注意：**这里 `frame_id: right/arm_base` 是给我们自己的
   `ee_pose_ik_controller` 看的**，节点内部会先转到 arm_base
   系，再 `_arm_base_pose_to_world` 正向变换到 `world` 系，
   最后才发给 MoveIt。**直接给 `/compute_ik` 传
   `frame_id: right/arm_base` 一定会 `FRAME_TRANSFORM_FAILURE`**
   （因为 Isaac 发的 `/tf` 上是 `right_arm_base` 不是
   `right/arm_base`），这一段的解释见上文 `_solve_moveit` 注释。

   IK 终端应在 5–15 ms 内打绿色 `succeeded`。如果显示 60+ ms，
   多半是 MoveIt 内部第一次加载，第二次开始就快。

6. 边界扫描复测（同 `15_debug_IK.md` 验证流程第 4 步）：沿世界
   z 轴每 1 cm 抬一次目标，记录第一次连续失败 ≥3 帧的 z 值。
   预期相对 cuRobo 边界明显外扩。

7. SpaceMouse 复测（同 `8_debug_teleop.md` 流程）：60 s 5 个动
   作，记录 `failed/total` 比例。预期失败比例从 cuRobo 的
   ~30% 降到 < 5%。

8. 抓瓶任务复测：

   ```bash
   ros2 run mobile_aloha_isaac_control right_bottle_pick_place_debug
   ```

   关注 `13_add_friction.md` 的高摩擦抓持有没有受影响（不应
   该受影响，因为只换 IK，不动场景层）。

## 在 RViz2 里拖 marker 发目标

我们没有走 MoveIt 的 `MotionPlanningDisplay` 来发目标 pose，
原因和这套架构对 controller 的要求有关：

1. `MotionPlanningDisplay` 自己**不**把目标 pose 发到任何
   topic，它是直接调 `/move_group/plan_kinematic_path` action
   并用自带的 controller_manager 执行。
2. 我们这套是直接出 `/isaac_joint_commands` 给 Isaac, 没有
   `controller_manager` + `JointTrajectoryController`,
   `MotionPlanningDisplay` 的 "Execute" 没法用。

所以专门写了 `rviz_ee_target_marker` 节点：每条手臂在 RViz 里
挂一个 6-DOF 拖拽 marker, 拖完直接 publish
`PoseStamped(frame_id=world)` 到 `/<side>_ee_target_pose`,
跟现有 SpaceMouse / debug teleop 走同一条 IK 链路.

> **API 命名坑**: `interactive_markers` 在 ROS2 Humble 上的 Python
> 绑定**保留 C++ camelCase**, 不是 ROS2 那套 snake_case PEP8.
> 必须写 `server.applyChanges()` / `server.setPose()` /
> `menu_handler.reApply()`; `apply_changes` 直接 `AttributeError`,
> 节点起不来 RViz 里就一颗球都看不见. `server.insert(...)` 和
> `menu_handler.insert(...)` / `menu_handler.apply(...)` 这几个
> 反而是小写 (因为是单词不是 compound word). 不要相信 IDE 的
> "PEP8 重命名" 提示.

启动方式：

```bash
ros2 launch mobile_aloha_isaac_control mobile_aloha_control.launch.py \
  start_rviz_target_marker:=true \
  start_rviz:=true \
  rviz_target_marker_sides:=both
```

`start_rviz:=true` 会用 `mobile_aloha_isaac_control/config/rviz/aloha_moveit.rviz`
打开 RViz2, 里面预置了:

- `Grid` + `TF` + `RobotModel` (URDF 来自 move_group 的
  `/robot_description` topic, 所以必须 `start_move_group:=moveit`
  或 `true`).
- `InteractiveMarkers`, namespace `/rviz_ee_target_marker`,
  会在每只手末端各显示一个**小**球 (中心 BUTTON, 用来右键弹
  菜单), 球外围有 RViz 自动画的 6 个 6-DOF 手柄: 3 条沿 X/Y/Z
  的双向**箭头**拖平移, 3 个绕 X/Y/Z 的**环**拖旋转. 每个手柄
  都加了 `always_visible=True`, 不需要 hover 也常驻显示.
  - 默认 `marker_scale_m=0.30 m`. 太小 (e.g. 0.15) 远视角根
    本看不见也抓不住, 太大又会把 ghost arm 全挡住, 0.30 在
    mobile ALOHA 这台机器人尺度下肉眼好找好抓. 想再大点的话
    launch 时加 `rviz_target_marker_scale_m:=0.5` (如果以后
    决定把这个参数从 yaml 提到 launch arg). 临时改可以直接
    动 `mobile_aloha_isaac_control/mobile_aloha_isaac_control/rviz_ee_target_marker.py`
    里 `declare_parameter("marker_scale_m", 0.30)` 那行的默认值.
  - **不要在中心 sphere 之外再加大块 visual** (例如方向箭头).
    如果 visual 大于 6-DOF 手柄的尺寸, RViz 把环和箭头画在 visual
    内部时会被遮住, 表面看就是 "看到一颗球但拖不动". 这条踩过.
- `Pose` display × 2, 显示 `/left_ee_current_pose_in_world`
  和 `/right_ee_current_pose_in_world`, 让你直观看到"实际
  EE"和"target marker"的差.
- `MotionPlanning` display 默认是关的, 想看可达性着色 (拖动时
  自动调 `/compute_ik` 给 ghost arm 上色) 就在 RViz 里勾上
  即可; "Plan" / "Execute" 按钮不要按, 没有 controller 管.

操作要点：

- **首次启动**: marker 先用一个写死的"arm_base 前 30 cm,
  z=1.4 m" 默认位置. 等 `/<side>_ee_current_pose_in_world`
  到第一帧时, marker 会自动 snap 到当前 EE 位姿一次, 之后
  就**不再**自动跟随真实 EE (否则会跟 IK 输出形成回路, 你拖
  到哪 marker 都会被拽回来).
- **手动重新对齐**: 在 marker 上右键 → `Snap to current EE
  pose`, 把 marker 回到当前 EE 位置. 用于"拖飞了想从头来过"
  的场景.
- **临时禁止发 target**: 右键 → `Pause publishing`. 拖动期间
  不会再 publish 到 `/<side>_ee_target_pose`. `Resume publishing`
  恢复.
- **publish 节奏**: `rviz_target_marker_publish_rate_hz`
  (默认 12 Hz, 跟 SpaceMouse 那条路一样), 拖动期间按这个频
  率 throttle, 鼠标松开时再补发一帧最终位姿, 保证 IK 拿得到
  最后定位.

注意：`move_group.launch.py` 里把 `publish_robot_description`
打开了 (transient_local), RViz2 RobotModel display 才能从
`/robot_description` 拿到 URDF. 不开的话 RViz 显示空场景.
Isaac Sim 不订阅这个 topic, 没有冲突.

### URDF / 关节名 / mesh 加载这套坑

`start_rviz:=true` 起 RViz 的时候, `mobile_aloha_control.launch.py`
会**额外**起两个节点把以下三个坑全堵上:

1. **URDF 用 `_isaac_material` 那版**, 不是普通的
   `split_aloha_rslidar_with_piper.urdf`. Isaac Sim USD 实际就是按
   `_isaac_material` 这版导入的, prim 名都叫
   `/World/split_aloha_rslidar_with_piper_isaac_material`. 两份 URDF
   的 `link/joint` 名 100% 一样, 只差 `<robot name>` 和 `<visual>`
   里的 `<material>` 标签. 走 `_isaac_material` 这版可以彻底解决
   `Semantic description is not specified for the same robot as the URDF`
   这条 SRDF 装载失败. 同步改的位置:
   - `mobile_aloha_moveit_config/launch/move_group.launch.py:default_urdf`
   - `mobile_aloha_moveit_config/config/split_aloha.srdf:<robot name>`
   - `mobile_aloha_isaac_control/launch/mobile_aloha_control.launch.py:_DEFAULT_URDF_PATH`
   - `mobile_aloha_isaac_control/config/curobo/mobile_aloha_{left,right}.yml:urdf_path`

2. **`package://` → `file:///` 而**不是**裸的 `/workspace/...`**: 这是
   一个非常容易踩的坑. RViz / MoveIt 加载 mesh 走
   `resource_retriever`, 内部用 libcurl 解析 URI. 如果把 `package://`
   替换成裸路径 `/workspace/isaac-sim/...`, libcurl 会把它当成
   "没 scheme 的相对 URL", 然后报一条**完全误导性的**
   `Could not load resource [/workspace/...]: Unable to open file ...`,
   就好像挂载点丢了一样, 实际上文件在那好好放着. 改成
   `file:///workspace/isaac-sim/...` 就立刻好. `_sanitize_urdf`
   的替换表:

   ```python
   _URDF_PACKAGE_REPLACEMENTS = {
       "package://split_aloha_mid_360":
           "file:///workspace/isaac-sim/mobile_aloha_sim/split_aloha_mid_360",
       "package://piper_description":
           "file:///workspace/isaac-sim/mobile_aloha_sim/piper_description",
   }
   ```

   注意 `ee_pose_ik_controller._sanitize_urdf_package_paths` (cuRobo
   专用那条) 仍然用裸路径, 因为 cuRobo 走 urdf_parser 不经
   resource_retriever, 也不需要加载 mesh (只用 collision spheres).

3. **Piper 链接的 `.STL` mesh 替换为 `.dae`**: 原 URDF 在 collision
   / visual 里大量引用
   `package://piper_description/meshes/<X>.STL`, RViz2 + Assimp
   在 ROS2 Humble 上对部分 piper STL 解析就直接报
   `Could not load resource [...top_camera_link.dae]: Unable to open file`
   这种(看着像找不到, 其实是 Assimp 解析失败). `meshes/dae/<X>.dae`
   下都有同名 dae 文件, 在 `_sanitize_urdf` 里加了一条正则替换:

   ```python
   _PIPER_STL_TO_DAE = re.compile(
       r"/workspace/isaac-sim/mobile_aloha_sim/piper_description/meshes/"
       r"([^/\"]+)\.STL"
   )
   ```

   `move_group.launch.py` 和 `mobile_aloha_control.launch.py` 都会
   各自跑一次, 保证发到 `/robot_description` 和喂给
   `robot_state_publisher` 的 URDF 都是 dae 版.

4. **`joint_state_name_bridge` 解决关节名不匹配**: Isaac Sim 把
   URDF 里的 `left/joint1` 写进 USD 时 `/` 不合法所以变
   `left_joint1`, 发出的 `/joint_states` 和 `/tf` 也都是下划线名
   (`left_arm_base`, `left_link6`). 但 URDF / SRDF / MoveIt 内部
   全是斜线名, 这条不匹配会同时打掉 3 个下游:

   - `robot_state_publisher` 拿到对不上 URDF 的 joint_states,
     不会发出斜线名的 TF, RViz `RobotModel` display 看到 URDF
     但找不到对应 frame, 机械臂原地不动.
   - `move_group` 的 `current_state_monitor` 刷屏
     `Joint 'left_joint1' not found in model`, 同时 RViz 的
     MotionPlanning Panel 按 Plan 就立刻报
     `Failed to fetch current robot state`.

   `joint_state_name_bridge` 把 `/joint_states` 里的
   `(left|right)_joint(\d+)` 改成 `\1/joint\2`, 发到
   `/joint_states_for_rviz`. 其它 `lifting_joint`、`fl_wheel`
   之类透传. 然后:

   - `mobile_aloha_control.launch.py` 里这个节点**不带
     IfCondition**, 默认无条件起 (它本身极轻).
   - `robot_state_publisher` (在 `start_rviz=true` 时拉起) 用
     `_isaac_material` URDF + `joint_states := /joint_states_for_rviz`
     remap, 发出 `base_link -> lifting_link -> left/arm_base ->
     left/link1 -> ...` 这棵斜线名 TF. Isaac 那棵下划线名 TF
     共存不冲突, RViz `world` Fixed Frame 之下 `world -> base_link`
     走 Isaac, 之后切到 rsp.
   - `mobile_aloha_moveit_config/launch/move_group.launch.py` 里
     `move_group` 的 `joint_states` 也 remap 到
     `/joint_states_for_rviz` (老版本是 remap 到 dummy topic 压
     missing joint 报错, 但那样 MotionPlanning 没法 Plan).

另外, RViz 的 MotionPlanning 面板要在自己 node 命名空间里能读到
`robot_description / _semantic / _kinematics`, 否则会刷
`No active joints or end effectors found for group 'right_arm'.
Make sure that kinematics.yaml is loaded in this node's namespace.`
这条警告. `mobile_aloha_control.launch.py` 把这三个 dict 都作
为 parameters 传给了 `rviz2` Node, 不依赖话题.

## 仍未解决的问题

迁完之后下面这些还和迁 MoveIt 无关，需要单独处理：

- **SpaceMouse → TCP 目标姿态映射"对不齐"**。继续在
  `mobile_aloha_debug_teleop.py` 内调 `rotation_frame: body|world`
  和 `space_*_sign`，参考上一轮对话和当前文件 117–123 行的参
  数语义。
- **升降柱在桌子附近的幽灵碰撞**。按 `14_debug_ghost_collision.md`
  的方案 A/B/C 处理。MoveIt 这条链路不开启 self-collision，
  所以场景层 collider 仍是唯一来源。
- **SpaceMouse 持续推送目标到不可达**。这是上层"目标限幅"问
  题。可以在 `mobile_aloha_debug_teleop` 里加一段：连续 IK
  failed N 帧时把 SpaceMouse 增量打折或暂停，等 IK 恢复
  succeeded 再继续。这一项写在 `15_debug_IK.md` 方案 E 里，
  不再重复。

## 何时再升到 B/C

A 模式跑稳之后再考虑：

- **B（IK + Servo）**：等需要"按住手柄实时跟随，无需 step 平
  滑节点"时，把 `joint_command_interpolator` 关掉，改用
  `MoveIt Servo` 输出 `/servo/delta_twist_cmds`，再写一个
  `JointTrajectoryController` ↔ `/isaac_joint_commands` 的小
  bridge。
- **C（完整 MoveIt + OMPL）**：等需要"先规划再执行"和
  RViz MotionPlanning Panel 时再上。这一步必然牵动
  `controllers.yaml`、`ros2_control` 配置、和 Isaac 端的
  `JointTrajectory` subscriber，工作量在一周量级。

## 与已有文档的关系

- `6_IMport_IK.md`：cuRobo 第一次接通的最小流程。本文是它的
  "替换后端"版本。两条路径在
  `dev_ws/ros2_ws/src/mobile_aloha_isaac_control/config/mobile_aloha_joints.yaml`
  里通过 `backend: curobo|moveit` 切换共存，需要回滚到 cuRobo
  时只改这一处即可。
- `7_debug_Ik.md`：红色方块拖拽 → `/right_ee_target_pose` 路径。
  迁 MoveIt 之后链路完全不变，只是 `right_ee_pose_ik_controller`
  内部的求解器换了。
- `8_debug_teleop.md`：键盘 + SpaceMouse 控制链路，本文不动它。
  SpaceMouse 旋转感问题在那条文档单独跟。
- `15_debug_IK.md`：cuRobo 调参方案。本文相当于 15 号方案的
  "升级路径"，但仅在 cuRobo 仍不够稳时才走；如果 15 号方案
  做完命中率就够，可以不迁。
- 改动范围：

  ```text
  dev_ws/ros2_ws/src/mobile_aloha_moveit_config/                     # 新增
  dev_ws/ros2_ws/src/trac_ik/                                        # 新增 (源码克隆)
  dev_ws/ros2_ws/src/mobile_aloha_isaac_control/config/mobile_aloha_joints.yaml
  dev_ws/ros2_ws/src/mobile_aloha_isaac_control/mobile_aloha_isaac_control/ee_pose_ik_controller.py
  dev_ws/ros2_ws/src/mobile_aloha_isaac_control/launch/mobile_aloha_control.launch.py
  dev_ws/scripts/run_ros2_humble_docker.sh                           # 镜像默认值
  ```

  恢复 cuRobo 不需要删任何文件，把 `backend` 切回 `curobo`、
  `start_move_group:=false` 即可。`src/trac_ik/` 留着不会被
  cuRobo 路径加载，MoveIt config 包同理。
