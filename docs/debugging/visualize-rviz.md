# RViz 可视化 Mobile ALOHA

本文记录如何在 RViz2 里订阅 Isaac Sim 发布的 `/joint_states`，并加载 Mobile ALOHA 的 URDF 显示整机模型。

## 原理

RViz 的 `RobotModel` 不能直接只靠 `/joint_states` 显示机器人。完整链路是：

```text
Isaac Sim
  -> /joint_states
  -> joint state name bridge
  -> /joint_states_rviz
  -> robot_state_publisher + Mobile ALOHA URDF
  -> /tf + /robot_description
  -> RViz RobotModel
```

其中：

- `/joint_states` 来自 Isaac Sim 场景里的 `ROS2PublishJointState`。
- joint state name bridge 把 Isaac 关节名转换成 URDF 关节名，例如 `left_joint1 -> left/joint1`、`right_joint2 -> right/joint2`。
- `robot_state_publisher` 读取 URDF，根据 `/joint_states_rviz` 发布各 link 的 TF。
- RViz 读取 `/robot_description` 和 `/tf`，显示机器人模型。

## 启动 Isaac Sim

宿主机：

```bash
cd /home/xiangpc/dataset/isaac-sim
./dev_ws/scripts/run_isaac_mobile_aloha.sh
```

打开：

```text
assets/5.mobile_aloha_in_market_ros_camera.usda
```

然后点击 Play。

## 启动 ROS2 控制

如果只想验证 RViz 和 `/joint_states`，可以关闭 IK 和 debug teleop：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  start_base_control:=false \
  start_ee_pose_ik:=false \
  start_debug_teleop:=false
```

正常情况下，Isaac Play 后应能看到：

```bash
ros2 topic list | grep -E "joint_states|tf|clock"
```

至少应包含：

```text
/joint_states
/tf
```

如果要确认反馈频率：

```bash
ros2 topic hz /joint_states
```

## 启动 RViz

宿主机另开终端：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
./scripts/run_mobile_aloha_rviz.sh
```

这个脚本会：

1. 进入正在运行的 ROS2 Humble Docker；如果没有运行，就启动一个。
2. 临时生成 `/tmp/mobile_aloha_rviz.urdf`，把 URDF 里的 `package://` mesh 路径替换成容器内 `file:///workspace/...` 路径；缺失的 Piper `.STL` collision mesh 会映射到已有的 `meshes/dae/*.dae`。
3. 启动 joint state name bridge，把 `/joint_states` 转成 `/joint_states_rviz`。
4. 启动 `robot_state_publisher`，订阅 `/joint_states_rviz`。
5. 打开 RViz，并加载 ROS2 专用配置：

```text
mobile_aloha_sim/split_aloha_mid_360/rviz/display_ros2.rviz
```

不要直接加载旧的 `display.rviz`。那个文件使用 ROS1 风格插件名，例如 `rviz/Grid`、`rviz/RobotModel`；RViz2 需要 `rviz_default_plugins/Grid`、`rviz_default_plugins/RobotModel`。

RViz 默认 fixed frame 是 `base_link`。

## 检查

在容器或宿主机 ROS2 环境里检查：

```bash
ros2 topic info /joint_states
ros2 topic info /joint_states_rviz
ros2 topic info /robot_description
ros2 topic info /tf
```

正常状态：

- `/joint_states` 有 Isaac publisher。
- `/joint_states_rviz` 有 name bridge publisher。
- `/robot_description` 有 `robot_state_publisher` publisher。
- `/tf` 有 `robot_state_publisher` publisher。
- RViz 左侧 `RobotModel` 状态为 OK。

## 常见问题

### `/joint_states` 没有消息

如果出现：

```text
WARNING: topic [/joint_states] does not appear to be published yet
```

优先检查：

- Isaac Sim 是否点击 Play。
- 是否打开的是 `assets/5.mobile_aloha_in_market_ros_camera.usda`。
- `isaacsim.ros2.bridge` 是否启用。
- 宿主机、Docker 和 Isaac 的 `ROS_DOMAIN_ID` / `RMW_IMPLEMENTATION` 是否一致。

### RViz 里没有机器人

检查：

```bash
ros2 topic echo --once /robot_description
ros2 topic echo --once /joint_states
ros2 topic echo --once /tf
```

如果 `/robot_description` 没有，说明 `robot_state_publisher` 没启动成功。

如果 `/joint_states` 没有，说明 Isaac 没在发布关节状态。

如果 RobotModel 里 mesh 丢失，检查脚本生成的 `/tmp/mobile_aloha_rviz.urdf` 中 mesh 路径是否存在，路径应以 `file:///workspace/isaac-sim/...` 开头。

### TF 重复警告

Isaac 场景本身也可能发布部分 `/tf`。同时启动 `robot_state_publisher` 时，RViz 或终端可能看到重复 TF publisher 的警告。

用于 RViz 模型调试时可以先忽略；真正需要干净 TF 树时，应只保留一个 TF 来源。
